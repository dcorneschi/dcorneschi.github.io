# Troubleshoot Custom NodePool and NodeClass in EKS Auto Mode

How to diagnose and fix provisioning issues with custom NodePools and NodeClasses in Amazon EKS Auto Mode — NotReady states, empty status, incorrect labels, and IAM permission errors.

When creating custom NodePools or NodeClasses in EKS Auto Mode, they can get stuck in "Unknown", "NotReady", or "Empty" states. This is usually caused by configuration errors in the NodePool spec, NodeClass reference, label restrictions, or missing IAM permissions on the node role.

## Issue: Custom NodePool Not Ready

```bash
kubectl get nodepool
```

```
NAME              NODECLASS   NODES   READY   AGE
custom-nodepool   default1    0       False   39s
general-purpose   default     0       True    15m
system            default     1       True    15m
```

### Cause 1: NodeClass Reference Not Found

The NodeClass referenced in the NodePool doesn't exist or is misspelled.

**Diagnose:**

```bash
kubectl describe nodepool <nodepool-name>
```

Look for `NodeClassNotFound` in the events:

```
Events:
  Type     Reason               Age    From       Message
  ----     ------               ----   ----       -------
  Normal   NodeClassReady       2m27s  karpenter  Status condition transitioned, Type: NodeClassReady, Status: Unknown -> False, Reason: NodeClassNotFound, Message: NodeClass not found on cluster
  Normal   Ready                2m27s  karpenter  Status condition transitioned, Type: Ready, Status: Unknown -> False, Reason: UnhealthyDependents, Message: NodeClassReady=False
  Warning                       20s    karpenter  Failed resolving NodeClass
```

**Fix:** Update the NodeClass name in the NodePool spec:

```bash
kubectl edit nodepool <nodepool-name>
```

Correct the `nodeClassRef`:

```yaml
nodeClassRef:
  group: eks.amazonaws.com
  kind: NodeClass
  name: <correct-nodeclass-name>
```

### Cause 2: Unsupported Labels (Karpenter Labels Instead of EKS Auto Mode Labels)

EKS Auto Mode uses labels prefixed with `eks.amazonaws.com` — not the standard Karpenter labels (`karpenter.k8s.aws`). Using Karpenter-style labels causes validation failure.

**Diagnose:**

```bash
kubectl describe nodepool <nodepool-name>
```

Look for `NodePoolValidationFailed` with a message about restricted labels:

```
Message: invalid value: label is restricted; specify a well known label or a custom label that does not use a restricted domain
(label=karpenter.k8s.aws/instance-category, restricted-labels=[k8s.io karpenter.k8s.aws karpenter.sh kubernetes.io], ...)
```

**Fix:** Replace Karpenter-style labels with their EKS Auto Mode equivalents:

| Karpenter Label | EKS Auto Mode Label |
|-----------------|---------------------|
| `karpenter.k8s.aws/instance-category` | `eks.amazonaws.com/instance-category` |
| `karpenter.k8s.aws/instance-family` | `eks.amazonaws.com/instance-family` |
| `karpenter.k8s.aws/instance-size` | `eks.amazonaws.com/instance-size` |
| `karpenter.k8s.aws/instance-cpu` | `eks.amazonaws.com/instance-cpu` |
| `karpenter.k8s.aws/instance-memory` | `eks.amazonaws.com/instance-memory` |
| `karpenter.k8s.aws/instance-generation` | `eks.amazonaws.com/instance-generation` |

Full list of supported labels: [EKS Auto Mode supported labels](https://docs.aws.amazon.com/eks/latest/userguide/create-node-pool.html#auto-supported-labels)

## Issue: Custom NodePool Status is Empty

```bash
kubectl get nodepool
```

```
NAME              NODECLASS   NODES   READY   AGE
eks-auto-mode     default                     8s
general-purpose   default     0       True    25m
```

The READY and NODES columns are blank — the NodePool isn't being processed at all.

### Cause: Incorrect Group or Kind in NodeClassRef

**Diagnose:**

```bash
kubectl describe nodepool <nodepool-name>
```

Check the `Node Class Ref` section:

```
Node Class Ref:
    Group:  karpenter.k8s.aws       # Wrong!
    Kind:   EC2NodeClass             # Wrong!
    Name:   default
```

**Fix:** In EKS Auto Mode, the NodeClassRef must use:

```yaml
nodeClassRef:
  group: eks.amazonaws.com    # Not karpenter.k8s.aws
  kind: NodeClass             # Not EC2NodeClass
  name: default
```

To fix in-place:

```bash
# Edit the nodepool
kubectl edit nodepool <nodepool-name>

# If the edit creates a temp file due to validation issues, force-apply it:
kubectl apply -f /tmp/kubectl-edit-XXXXXXX.yaml --force
```

## Issue: Custom NodeClass in NotReady State

```bash
kubectl get nodeclass
```

```
NAME           ROLE                    READY   AGE
customclass    AutoModeCustomRole      False   6m29s
default        AmazonEKSAutoNodeRole   True    41h
```

### Cause: Node IAM Role Missing Permissions or Access Entry

The custom node IAM role lacks the required EKS permissions to join nodes to the cluster.

**Diagnose:**

```bash
kubectl describe nodeclass customclass
```

Look for `UnauthorizedNodeRole`:

```
Events:
  Type    Reason                Age   From       Message
  ----    ------                ----  ----       -------
  Normal  InstanceProfileReady  50m   karpenter  Status condition transitioned, Type: InstanceProfileReady, Status: Unknown -> False, Reason: UnauthorizedNodeRole, Message: Role AutoModeCustomRole is unauthorized to join nodes to the cluster
  Normal  Ready                 50m   karpenter  Status condition transitioned, Type: Ready, Status: Unknown -> False, Reason: UnhealthyDependents, Message: ValidationSucceeded=False, InstanceProfileReady=False
```

Also check CloudTrail for `Unauthorized` errors filtered by `eks.amazonaws.com`.

**Fix:** The custom node role needs:

1. **Minimum IAM permissions** — see [Amazon EKS Auto Mode node IAM role](https://docs.aws.amazon.com/eks/latest/userguide/auto-create-node-role.html)
2. **An EKS Access Entry** with the `AmazonEKSAutoNodePolicy` access policy

EKS automatically creates an access entry for the default node role, but custom roles need a manually created one:

```bash
aws eks create-access-entry \
  --cluster-name <cluster-name> \
  --principal-arn arn:aws:iam::<account-id>:role/AutoModeCustomRole \
  --type EC2_LINUX

aws eks associate-access-policy \
  --cluster-name <cluster-name> \
  --principal-arn arn:aws:iam::<account-id>:role/AutoModeCustomRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSAutoNodePolicy \
  --access-scope type=cluster
```

> For more on access entries, see [Grant IAM users access to Kubernetes with EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html).

## Quick Reference: NodePool/NodeClass Troubleshooting

| Symptom | `kubectl describe` Signal | Root Cause | Fix |
|---------|---------------------------|-----------|-----|
| NodePool READY=False | `NodeClassNotFound` | NodeClass name misspelled or missing | Fix `nodeClassRef.name` |
| NodePool READY=False | `NodePoolValidationFailed` with restricted labels | Using `karpenter.k8s.aws/*` labels | Replace with `eks.amazonaws.com/*` labels |
| NodePool status empty | `Group: karpenter.k8s.aws, Kind: EC2NodeClass` | Wrong API group/kind | Use `group: eks.amazonaws.com, kind: NodeClass` |
| NodeClass READY=False | `UnauthorizedNodeRole` | Missing IAM permissions or access entry | Add IAM policy + create EKS access entry |

## Related Resources

- [EKS Auto Mode — create custom node pools](https://docs.aws.amazon.com/eks/latest/userguide/create-node-pool.html)
- [Amazon EKS Auto Mode node IAM role](https://docs.aws.amazon.com/eks/latest/userguide/auto-create-node-role.html)
- [Grant IAM users access to Kubernetes with EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- [Troubleshoot EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/auto-troubleshoot.html)
