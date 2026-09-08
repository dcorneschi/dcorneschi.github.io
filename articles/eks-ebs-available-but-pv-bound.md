# EBS Volume Available but Kubernetes PV Still Bound

## The Problem

An EBS volume shows as **available** (detached) in AWS, but the corresponding
Kubernetes PersistentVolume (PV) is still in **Bound** status. This is an
inconsistent state — Kubernetes believes the volume is in use, but AWS shows it's
not attached to any instance.

## Common Causes

### 1. Pod is not running

The pod using the PVC is scaled to 0, stuck in `Pending`, or has crashed. The
volume detaches from the node, but the PV/PVC binding stays intact because the
claim still exists.

### 2. Node was terminated

The node the pod ran on was replaced (spot interruption, ASG scale-in, node
drain) and the replacement pod hasn't scheduled yet or is stuck `Pending`.

### 3. Stuck VolumeAttachment

Kubernetes still has a `VolumeAttachment` object referencing the old node, but
the AWS-side attachment was force-detached or timed out. The CSI driver won't
reattach the volume to a new node because it thinks it's still attached
elsewhere.

### 4. AZ mismatch

The EBS volume is in one Availability Zone but no suitable node exists in that AZ
to schedule the pod. The pod stays `Pending` and the volume stays detached. (EBS
volumes can only attach to instances in the same AZ.)

## Investigation Commands

### Find the PVC and namespace from the PV

```bash
kubectl get pv <pv-name> -o jsonpath='Namespace: {.spec.claimRef.namespace} | PVC: {.spec.claimRef.name}{"\n"}'
```

### Check whether any pod is using that PVC

```bash
kubectl get pods -n <namespace> -o json | jq -r \
  '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName == "<pvc-name>") | "\(.metadata.name) -> status: \(.status.phase)"'
```

No output means no pod references the PVC.

### Check pod status (if one exists)

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for events like `FailedAttachVolume`, `FailedScheduling`, or
`Multi-Attach error`.

### Check VolumeAttachment objects

```bash
kubectl get volumeattachment | grep <pv-name>
```

A stale `VolumeAttachment` pointing to a non-existent node is a strong indicator
of cause #3.

```bash
# Inspect the attachment — check .spec.nodeName
kubectl get volumeattachment <attachment-name> -o yaml
```

If that node no longer exists, the attachment is stale.

### Describe the PV for events

```bash
kubectl describe pv <pv-name>
```

### Verify the EBS volume's AZ and state

```bash
aws ec2 describe-volumes --volume-ids <vol-id> \
  --query 'Volumes[0].{AZ:AvailabilityZone,State:State,Size:Size}'
```

### Check whether nodes exist in that AZ

```bash
kubectl get nodes -l topology.kubernetes.io/zone=<az> --no-headers
```

## Resolution

### Pod scaled to zero (expected)

No action needed — the volume reattaches automatically when the pod scales back up.

### Pod stuck Pending (scheduling issues)

Fix the root cause: node affinity/taints blocking scheduling, insufficient
resources in the target AZ, or no nodes in the volume's AZ.

```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 Events
```

### Stale VolumeAttachment

Delete the stale attachment so the CSI driver can reattach the volume to the
correct node:

```bash
# Confirm the node referenced by the attachment no longer exists
kubectl get node <node-name-from-attachment>

# If it's gone, delete the stale attachment
kubectl delete volumeattachment <attachment-name>
```

The pod should then attach the volume to its current node.

> Only delete a `VolumeAttachment` after confirming the volume is genuinely
> detached in AWS (`State: available`) and the referenced node is gone. Deleting
> one for a volume still attached elsewhere can cause data issues.

### Workload is gone — clean up

If the workload no longer exists and the volume isn't needed:

```bash
# Delete the PVC (triggers the PV's reclaim policy)
kubectl delete pvc <pvc-name> -n <namespace>

# If reclaim policy is Retain, delete the PV manually
kubectl delete pv <pv-name>

# Snapshot first, then delete the EBS volume in AWS
aws ec2 create-snapshot --volume-id <vol-id> --description "Backup before deletion"
aws ec2 delete-volume --volume-id <vol-id>
```

## Prevention

- Use the `WaitForFirstConsumer` volume binding mode so a PV is provisioned in
  the AZ where the pod actually schedules (avoids AZ mismatch).
- Alert on PVs that stay `Bound` with no running pod for an extended period.
- Use PodDisruptionBudgets and proper node-drain procedures to avoid orphaned
  attachments.
- Consider a `Delete` reclaim policy for non-critical workloads so unused volumes
  don't accumulate.

## Related

- [Persistent Volumes on EKS with EBS CSI Driver](articles/eks-persistent-volumes-ebs-csi.md)
- [Dynamic PV/PVC Provisioning with a StorageClass](articles/kubernetes-dynamic-provisioning-storageclass.md)
- [Find Unattached (Available) EBS Volumes](articles/aws-ebs-unattached-volumes.md)

## Skills Practiced

- Diagnosing an AWS/Kubernetes state mismatch (EBS available vs PV Bound)
- Tracing a PV to its PVC and pod, and reading VolumeAttachment objects
- Clearing stale VolumeAttachments and understanding the AZ attachment constraint
- Safely cleaning up orphaned PVs/PVCs and EBS volumes with a snapshot first
