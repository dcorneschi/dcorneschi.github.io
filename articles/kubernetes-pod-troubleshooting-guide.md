# Kubernetes Pod Troubleshooting Guide

This guide covers troubleshooting pods — the applications you deploy into
Kubernetes — organized by the state a pod gets stuck in.

No matter which error you hit, the first step is almost always to get the pod's
current state (and events) and its logs:

```sh
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

The pod's events and logs are usually enough to identify the issue.

## Pod Stuck in Pending

`Pending` means the pod hasn't been scheduled yet. Check the events — they'll
show why:

```sh
$ kubectl describe pod mypod
...
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  12s (x6 over 27s)  default-scheduler  0/4 nodes are available: 2 Insufficient cpu.
```

Usually this is insufficient resources of one kind or another. Common causes:

- The cluster doesn't have enough CPU, memory, or GPU. Adjust the pod's resource
  requests or add nodes.
- The pod requests more than any single node's capacity. Lower the request or add
  larger nodes.
- The pod uses a `hostPort` that's already taken on candidate nodes. Use a
  Service instead.
- Node affinity, taints/tolerations, or topology constraints exclude all nodes.

## Pod Stuck in Waiting or ContainerCreating

Here the pod has been scheduled to a node but can't run yet. Again, start with
`kubectl describe pod <pod-name>`:

```sh
$ kubectl -n kube-system describe pod nginx-pod
Events:
  Type     Reason                 Age               From               Message
  ----     ------                 ----              ----               -------
  Normal   Scheduled              1m                default-scheduler  Successfully assigned nginx-pod to node1
  Normal   SuccessfulMountVolume  1m                kubelet, gpu13     MountVolume.SetUp succeeded for volume "config-volume"
  Warning  FailedSync             2s (x4 over 46s)  kubelet, gpu13     Error syncing pod
  Normal   SandboxChanged         1s (x4 over 46s)  kubelet, gpu13     Pod sandbox changed, it will be killed and re-created.
```

The sandbox can't start. Check kubelet logs on that node for the detail:

```sh
$ journalctl -u kubelet
...
E0314 04:22:04.649912 cni.go:294] Error adding network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.4.1/24
E0314 04:22:05.965801 remote_runtime.go:91] RunPodSandbox from runtime service failed: ... NetworkPlugin cni failed to set up pod "nginx-pod" network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.4.1/24
```

Here the `cni0` bridge has an unexpected IP. The simplest fix is to delete the
bridge (the CNI plugin recreates it as needed):

```sh
ip link set cni0 down
brctl delbr cni0        # or: ip link delete cni0 type bridge
```

That's one example (a network config issue). Other things that commonly go wrong
in this state:

- **Image pull failures** — wrong image name, unreachable/unauthenticated
  registry, image not pushed, missing pull secret, or timeouts on large images
  (tuning kubelet `--image-pull-progress-deadline` / `--runtime-request-timeout`
  can help).
- **Pod sandbox network setup** — CNI misconfiguration, or IP exhaustion in the
  pod CIDR.
- **Container fails to start** — wrong `command`/`args`, or a bad binary baked
  into the image.

## Pod Stuck in ImagePullBackOff

`ImagePullBackOff` means the image couldn't be pulled after several retries —
usually a wrong image name or a bad/missing registry credential. Verify with
`docker pull <image>` (or `crictl pull <image>`).

```sh
$ kubectl describe pod mypod
...
  Warning  Failed   14s   kubelet  Failed to pull image "a1pine": ... repository a1pine not found: does not exist or no pull access
  Warning  Failed   14s   kubelet  Error: ErrImagePull
  Normal   BackOff  4s    kubelet  Back-off pulling image "a1pine"
  Warning  Failed   1s    kubelet  Error: ImagePullBackOff
```

For private images, create a registry secret:

```sh
kubectl create secret docker-registry my-secret \
  --docker-server=DOCKER_REGISTRY_SERVER \
  --docker-username=DOCKER_USER \
  --docker-password=DOCKER_PASSWORD \
  --docker-email=DOCKER_EMAIL
```

Then reference it in the pod spec:

```yaml
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: my-secret
```

## Pod Stuck in CrashLoopBackOff

The pod started, then exited abnormally (its `restartCount` is > 0). Look at the
container logs:

```sh
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Logs from the previous crashed instance (most useful)
kubectl logs --previous <pod-name>
```

The logs usually reveal the cause: the process exited, a health check failed, or
it was OOMKilled.

```sh
$ kubectl describe pod mypod
...
Containers:
  sh:
    State:          Terminated
      Reason:       OOMKilled
      Exit Code:    2
    Restart Count:  3
    Limits:
      cpu:     1
      memory:  1G
    Requests:
      cpu:        100m
      memory:     500M
```

You can also run a command inside the container to inspect its logs directly:

```sh
kubectl exec <pod-name> -- cat /var/log/app/system.log
```

If none of that helps, find the pod's node and inspect kubelet/runtime logs there:

```sh
kubectl get pod <pod-name> -o wide     # find the node
ssh <username>@<node-name>
```

## Pod Stuck in Error

The pod was scheduled but failed to start. `kubectl describe pod` again. Common
reasons:

- Referencing a non-existent ConfigMap, Secret, or PV.
- Exceeding resource limits (e.g. a LimitRange).
- Violating a Pod Security policy.
- Not authorized to cluster resources (with RBAC, the service account needs a
  RoleBinding).

## Pod Stuck in Terminating or Unknown

Since Kubernetes v1.5, the controller manager doesn't delete pods just because a
node is NotReady — such pods are marked `Terminating` or `Unknown`. If you're
sure they're no longer wanted, there are three ways to remove them:

- **Delete the node** from the cluster: `kubectl delete node <node-name>`. With a
  cloud provider, the node is usually removed automatically once the VM is gone.
- **Recover the node.** After kubelet restarts, it reconciles pod status with the
  API server and restarts or deletes those pods.
- **Force delete:** `kubectl delete pod <pod> --grace-period=0 --force`. Not
  recommended unless you know what you're doing — for StatefulSet pods, force
  deletion can cause data loss or split-brain.

If even force deletion doesn't work, a stuck **finalizer** is often the cause.
Edit the pod and remove it:

```yaml
"finalizers": [
  "foregroundDeletion"
]
```

```sh
kubectl edit pod <pod-name>   # remove the finalizers entry
```

## Pod Runs but Doesn't Do What It Should

If the pod is running but not behaving as expected, there may be an error in the
manifest — a section nested incorrectly, or a mistyped key that gets silently
ignored.

Recreate with server-side validation:

```sh
kubectl delete pod mypod
kubectl apply --validate -f mypod.yaml
```

Or read back what was actually created and compare against what you intended:

```sh
kubectl get pod mypod -o yaml
```

## Static Pod Not Recreated After Manifest Change

Kubelet watches `/etc/kubernetes/manifests` (set by `--pod-manifest-path`) via
inotify. Occasionally it misses an event and a static pod isn't recreated after
you edit its manifest. Restarting kubelet resolves it:

```sh
sudo systemctl restart kubelet
```

## References

- [Troubleshoot Applications (Kubernetes docs)](https://kubernetes.io/docs/tasks/debug/debug-application/)

## Related

- [CrashLoopBackOff Explained](articles/kubernetes-crashloopbackoff-explained.md)
- [ImagePullBackOff Troubleshooting Guide](articles/kubernetes-imagepullbackoff-troubleshooting.md)
- [Troubleshooting CrashLoopBackOff with No Logs](articles/kubernetes-crashloopbackoff-no-logs.md)
- [Kubernetes Resource Scheduling & Node Capacity](articles/kubernetes-resource-scheduling-node-capacity.md)

## Skills Practiced

- Diagnosing pods by state — Pending, ContainerCreating, ImagePullBackOff, CrashLoopBackOff, Error, Terminating
- Reading pod events, container logs (including `--previous`), and node-level kubelet logs
- Fixing CNI bridge issues, image-pull auth, and stuck finalizers
- Validating manifests and handling static pod refresh
