# EKS Node Group Rolling Updates: How Nodes Are Recycled

When you update a launch template version on an EKS managed node group, EKS performs a surge-based rolling update to replace all nodes with the new configuration.

## Update Strategy

EKS follows this sequence for each node:

1. Launch a new node using the **new** launch template version
2. Wait for the new node to become `Ready`
3. Cordon the old node (mark unschedulable)
4. Drain the old node (evict pods, respecting PDBs and grace periods)
5. Terminate the old instance
6. Repeat until all nodes are on the new template

## Triggering the Update

```sh
# Update the node group to use a new launch template version
aws eks update-nodegroup-version \
  --cluster-name <cluster> \
  --nodegroup-name <nodegroup> \
  --launch-template name=<lt-name>,version=<new-version>
```

With Terraform, changing the `launch_template.version` in `aws_eks_node_group` triggers this automatically on apply.

## Controlling the Rollout

The `updateConfig` setting controls how many nodes can be replaced simultaneously:

```sh
# Set max unavailable (absolute number)
aws eks update-nodegroup-config \
  --cluster-name <cluster> \
  --nodegroup-name <nodegroup> \
  --update-config '{"maxUnavailable": 1}'

# Or as a percentage
aws eks update-nodegroup-config \
  --cluster-name <cluster> \
  --nodegroup-name <nodegroup> \
  --update-config '{"maxUnavailablePercentage": 33}'
```

| Setting | Effect |
|---------|--------|
| `maxUnavailable: 1` | Replace one node at a time (safest, slowest) |
| `maxUnavailable: 2` | Replace two nodes at a time |
| `maxUnavailablePercentage: 33` | Replace up to 33% of nodes simultaneously |

## What Happens Under the Hood

```
LT v1 nodes: [node-A, node-B, node-C]  (running)

1. EKS launches node-D with LT v2
2. node-D becomes Ready
3. EKS cordons node-A
4. EKS drains node-A (respects PDBs)
5. Pods reschedule to node-D (or B/C)
6. node-A terminates

7. EKS launches node-E with LT v2
8. Repeat for node-B...

9. EKS launches node-F with LT v2
10. Repeat for node-C...

LT v2 nodes: [node-D, node-E, node-F]  (running)
```

## Key Behaviors

- **PDBs are respected**: EKS won't evict pods if it would violate a PodDisruptionBudget. If a PDB blocks the drain, the update stalls.
- **DaemonSets are ignored**: They're expected on every node and are not drained.
- **Standalone pods are deleted**: Pods with no owner (not managed by a Deployment/StatefulSet/Job) are evicted and NOT rescheduled automatically.
- **Grace periods are honored**: Pods get their `terminationGracePeriodSeconds` before being killed.
- **New nodes must be Ready**: EKS waits for the replacement node to pass health checks before draining the old one.

## Force Update

If a drain is stuck (e.g., a PDB blocks eviction indefinitely), you can force it:

```sh
aws eks update-nodegroup-version \
  --cluster-name <cluster> \
  --nodegroup-name <nodegroup> \
  --launch-template name=<lt-name>,version=<new-version> \
  --force
```

> **Warning:** Force update deletes pods without respecting PDBs. Use only when the update is stuck and you accept potential disruption.

## Checking Update Status

```sh
# List updates for a node group
aws eks list-updates --name <cluster> --nodegroup-name <nodegroup>

# Describe a specific update
aws eks describe-update --name <cluster> --nodegroup-name <nodegroup> --update-id <update-id>
```

From kubectl:

```sh
# Watch nodes being cordoned and replaced
kubectl get nodes -w

# Check for pods stuck in Pending (waiting for capacity)
kubectl get pods -A --field-selector status.phase=Pending

# See which nodes are cordoned
kubectl get nodes -o custom-columns=NAME:.metadata.name,SCHEDULABLE:.spec.unschedulable
```

## What Triggers a Rolling Update

Any change to the node group that requires new instances:

| Change | Triggers Rolling Update |
|--------|------------------------|
| Launch template version | Yes |
| AMI ID (in launch template) | Yes |
| Instance type (in launch template) | Yes |
| Kubernetes version upgrade | Yes |
| Disk size change | Yes |
| Userdata change | Yes |
| Labels/taints change | No (applied in-place) |
| Scaling (desired count) | No (just adds/removes) |

## Self-Managed Nodes: The Difference

For self-managed nodes, **none of this is automatic**. When you change the launch template on an ASG:

- New instances launched by scale-out use the new template
- **Existing instances are NOT touched** — they keep running with the old template
- You must manually trigger an instance refresh or terminate old instances

```sh
# Trigger an ASG instance refresh (self-managed)
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name <asg-name> \
  --preferences '{"MinHealthyPercentage": 66, "InstanceWarmup": 300}'
```

But ASG instance refresh does NOT drain Kubernetes nodes — it just terminates instances. You need lifecycle hooks + Node Termination Handler to drain before termination.

| Capability | Managed Node Group | Self-Managed (ASG) |
|------------|-------------------|-------------------|
| Automatic rolling update | Yes | No |
| Respects PDBs | Yes | No (unless you add NTH) |
| Cordons before drain | Yes | No (manual) |
| Surge (new before old dies) | Yes | With instance refresh only |
| Force option | Yes | N/A |

## Best Practices

- **Always set PDBs** on critical workloads — they protect against too many pods being evicted at once
- **Use `maxUnavailablePercentage: 33`** for large node groups to speed up updates without too much risk
- **Monitor during updates** — watch for pods stuck in Pending, which indicates insufficient capacity
- **Test launch template changes** in a dev cluster first — a bad AMI or userdata will cause nodes to fail to join
- **Keep node groups small enough** that a rolling update completes in a reasonable time (50+ nodes can take hours with `maxUnavailable: 1`)
- **Use multiple node groups** to isolate blast radius — update one group at a time in production


## Terraform: Patching Nodes Per AZ with -target

In production, you often have one node group per availability zone. This lets you patch one AZ at a time, keeping the majority of your capacity available while nodes are being recycled.

### Node Group Per AZ Layout

```hcl
variable "azs" {
  default = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
}

resource "aws_launch_template" "workers" {
  name_prefix   = "eks-workers-"
  image_id      = var.ami_id
  instance_type = "m5.xlarge"

  user_data = base64encode(templatefile("userdata.tpl", {
    cluster_name = var.cluster_name
  }))

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "eks-worker"
    }
  }
}

resource "aws_eks_node_group" "workers" {
  for_each = toset(var.azs)

  cluster_name    = var.cluster_name
  node_group_name = "workers-${each.value}"
  node_role_arn   = aws_iam_role.workers.arn
  subnet_ids      = [var.subnet_ids[each.value]]

  scaling_config {
    desired_size = 3
    min_size     = 2
    max_size     = 5
  }

  launch_template {
    id      = aws_launch_template.workers.id
    version = aws_launch_template.workers.latest_version
  }

  update_config {
    max_unavailable = 1
  }

  tags = {
    AZ = each.value
  }
}
```

### Patching One AZ at a Time

When the AMI changes (e.g., new security patches), update the AMI ID in your Terraform variables and roll out per AZ. Terraform resolves dependencies automatically — targeting the node group will plan the launch template change along with it.

First, generate the target flags for a specific AZ from the plan output:

```sh
# Generate -target flags for all resources in a specific AZ
terraform plan | grep '# aws' | grep eu-west-1a | awk '{printf("-target=%s ",$3)} END {printf("\n")}' | sed -re 's/"/\\"/g'
```

Then apply per AZ:

```sh
# Step 1: Roll AZ-a
terraform apply $(terraform plan | grep '# aws' | grep eu-west-1a | awk '{printf("-target=%s ",$3)} END {printf("\n")}' | sed -re 's/"/\\"/g')

# Step 2: Verify nodes in AZ-a are on the new AMI and pods are healthy
kubectl get nodes -l topology.kubernetes.io/zone=eu-west-1a
kubectl get pods -A --field-selector status.phase!=Running,status.phase!=Succeeded

# Step 3: Roll AZ-b
terraform apply $(terraform plan | grep '# aws' | grep eu-west-1b | awk '{printf("-target=%s ",$3)} END {printf("\n")}' | sed -re 's/"/\\"/g')

# Step 4: Verify AZ-b
kubectl get nodes -l topology.kubernetes.io/zone=eu-west-1b
kubectl get pods -A --field-selector status.phase!=Running,status.phase!=Succeeded

# Step 5: Roll AZ-c
terraform apply $(terraform plan | grep '# aws' | grep eu-west-1c | awk '{printf("-target=%s ",$3)} END {printf("\n")}' | sed -re 's/"/\\"/g')

# Step 6: Verify AZ-c
kubectl get nodes -l topology.kubernetes.io/zone=eu-west-1c

# Step 7: Final plan to confirm no remaining drifts
terraform plan
```

This approach dynamically picks up ALL resources that changed for a given AZ — not just the node group, but also the launch template, security groups, or any other dependency that Terraform plans to update.

### Why Target Per AZ

| Approach | Risk | Duration |
|----------|------|----------|
| `terraform apply` (all at once) | All AZs roll simultaneously, potential capacity crunch | Fast but risky |
| `-target` per AZ | Only 1/3 of capacity rolling at a time | Slower but safe |
| `-target` per AZ with verification | Can stop if issues detected | Safest for production |

### Verification Between AZ Rolls

```sh
# Check all nodes in the updated AZ are on the new AMI
kubectl get nodes -l topology.kubernetes.io/zone=eu-west-1a \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.osImage}{"\n"}{end}'

# Check no pods are stuck
kubectl get pods -A | grep -v Running | grep -v Completed

# Check PDB status (make sure none are disrupted beyond allowed)
kubectl get pdb -A

# Check node group update status
aws eks describe-nodegroup \
  --cluster-name <cluster> \
  --nodegroup-name workers-eu-west-1a \
  --query "nodegroup.health" --output json
```

### Handling Failures

If a rolling update fails mid-way (e.g., new AMI has broken userdata):

```sh
# Check what went wrong
aws eks describe-nodegroup \
  --cluster-name <cluster> \
  --nodegroup-name workers-eu-west-1a \
  --query "nodegroup.{Status:status, Health:health}"

# Revert the AMI in variables
# Then apply only the launch template to create a new version
terraform apply -target=aws_launch_template.workers

# Re-roll the failed AZ with the fixed template
terraform apply -target='aws_eks_node_group.workers["eu-west-1a"]'
```

### Automation Script

```sh
#!/bin/bash
set -euo pipefail

CLUSTER="my-cluster"
AZS=("eu-west-1a" "eu-west-1b" "eu-west-1c")

for AZ in "${AZS[@]}"; do
  echo "Generating targets for $AZ..."
  TARGETS=$(terraform plan | grep '# aws' | grep "$AZ" | awk '{printf("-target=%s ",$3)} END {printf("\n")}' | sed -re 's/"/\\"/g')

  if [ -z "$TARGETS" ]; then
    echo "No changes for $AZ, skipping."
    continue
  fi

  echo "Rolling $AZ with targets: $TARGETS"
  eval terraform apply $TARGETS -auto-approve

  echo "Waiting for nodes in $AZ to be Ready..."
  sleep 30
  kubectl wait --for=condition=Ready nodes -l "topology.kubernetes.io/zone=$AZ" --timeout=600s

  echo "Checking for unhealthy pods..."
  UNHEALTHY=$(kubectl get pods -A --field-selector 'status.phase!=Running,status.phase!=Succeeded' --no-headers 2>/dev/null | wc -l)
  if [ "$UNHEALTHY" -gt 0 ]; then
    echo "WARNING: $UNHEALTHY unhealthy pods detected after rolling $AZ. Pausing."
    kubectl get pods -A --field-selector 'status.phase!=Running,status.phase!=Succeeded'
    exit 1
  fi

  echo "$AZ complete."
done

echo "All AZs rolled. Running final plan..."
terraform plan
```

### Notes

- Just change the AMI (or other config) in your Terraform files and target node groups one AZ at a time. Terraform resolves the launch template dependency automatically.
- The `latest_version` reference on the node group automatically picks up the new launch template version — no manual version pinning needed.
- Filtering by AZ with `kubectl get nodes -l topology.kubernetes.io/zone=<az>` lets you verify only the nodes that were just rolled.
- Use `terraform plan` (no target) at the end to confirm there are no remaining drifts.
- For large clusters, increase `max_unavailable` per node group to speed up individual AZ rolls while still keeping cross-AZ safety.
