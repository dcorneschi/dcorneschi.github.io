# "pull QPS exceeded": Kubelet Image-Pull Throttling

If pods fail to start during a busy scheduling event — a node coming up, a scale-out, or
a fleet being recycled for patching — you may see events like this:

```
Warning  Failed    52s   kubelet   Failed to pull image "a.b.c/image:0.1": pull QPS exceeded
Warning  Failed    52s   kubelet   Error: ErrImagePull
Normal   BackOff   45s   kubelet   Back-off pulling image "a.b.c/image:0.1"
Warning  Failed    45s   kubelet   Error: ImagePullBackOff
```

The key phrase is **`pull QPS exceeded`**. This is the **kubelet on the node** rate-limiting
how fast *it* pulls container images — it is refusing to start the pull, not the registry
rejecting you and not the Kubernetes API server throttling you.

> This is a different mechanism from the `Waited for ... due to client-side throttling,
> not priority and fairness` message. That one is `client-go` throttling calls to the
> **kube-apiserver**. This one is the **kubelet** throttling **image pulls** from a
> **registry**. They often show up together during node recycling, but the knobs and fixes
> are separate. See `kubectl-client-side-throttling.md`.

---

## What the message means

Each kubelet has a built-in **image-pull rate limiter**, controlled by two settings:

- **`registryPullQPS`** — sustained image pulls per second the kubelet will start.
  `0` means no limit.
- **`registryBurst`** — the size of the short-term bucket; pulls may temporarily burst to
  this number as long as the sustained rate stays within `registryPullQPS`. Only used when
  `registryPullQPS > 0`.

When more image pulls are requested in a short window than the limiter allows, the kubelet
rejects the excess with **`pull QPS exceeded`**, the pod goes to `ErrImagePull`, and the
kubelet backs off and retries (`ImagePullBackOff`). Because it retries with backoff, this
is often *transient* — the pod eventually starts once the burst drains — but under sustained
load it can stall startup for minutes.

### Defaults

Historically the defaults were low:

- `registryPullQPS: 5`
- `registryBurst: 10`

Newer Kubernetes raised these (to `20` / `40`) and flipped `serializeImagePulls` to allow
parallel pulls, but **many managed distros and existing nodes still run the old `5` / `10`
defaults**. On a node that has to pull a lot of distinct images at once, `5` QPS is easy to
blow past. Check what your nodes actually use rather than assuming — see below.

---

## Why it happens (and why recycling for patching triggers it)

The limiter is only a problem when a **lot of distinct image pulls are requested on one
node at nearly the same time**. Common triggers:

- **Nodes being recycled for patching / rolling replacement.** This is the classic case.
  A freshly patched (or newly created) node has an **empty image cache**, so *every*
  rescheduled pod must pull its image from the registry. When many pods land on the new
  node at once, their pulls all hit the kubelet limiter simultaneously.
- **Scale-out / cluster autoscaler / Karpenter.** New nodes start cold and pull everything
  from scratch.
- **Node failover.** Pods from a lost node reschedule en masse onto survivors, each needing
  its image.
- **Many containers per pod / many distinct images.** DaemonSets, sidecars, and init
  containers multiply the number of concurrent pulls.

Note the difference from `client-go` throttling: recycling for patching drives **both**
messages, but for different reasons — the API server sees a burst of control-plane calls
(client-side throttling), while each new node sees a burst of image pulls (`pull QPS
exceeded`).

> **Look past the limiter, too.** Sometimes `pull QPS exceeded` is a *symptom* rather than
> the root cause: if each pull is slow (disk I/O saturation on the node, a slow or
> throttled registry, huge images), pulls stay in flight longer and pile up against the
> limiter. Raising QPS/burst on a node that is disk- or network-bound just trades one
> failure for another. Check pull *duration* and node disk I/O if raising the limits does
> not help.

---

## How to confirm it is the kubelet limiter

1. **Read the event source.** The event is attributed to `kubelet` and the reason text is
   literally `pull QPS exceeded`. That is definitive — it is the node-side limiter, not the
   registry.

   ```bash
   kubectl describe pod <pod> | grep -A2 -i "pull QPS"
   ```

2. **Distinguish it from a registry rate limit.** Registry-side throttling looks different
   — e.g. Docker Hub returns `toomanyrequests: You have reached your pull rate limit` or an
   HTTP `429`, and ECR/GCR/etc. return their own auth/limit errors. If the message names a
   registry or says `toomanyrequests`, that is the registry, not the kubelet.

3. **Check the node's actual config** to see the values in force:

   ```bash
   # Via the kubelet's configz endpoint (through the API server proxy)
   kubectl get --raw "/api/v1/nodes/<node-name>/proxy/configz" | \
     grep -o '"registryPullQPS":[0-9]*\|"registryBurst":[0-9]*\|"serializeImagePulls":[a-z]*'
   ```

   Or inspect the kubelet config file on the node (path varies by distro), commonly
   `/etc/kubernetes/kubelet/kubelet-config.json` or `/var/lib/kubelet/config.yaml`.

---

## Fixes

### 1. Raise `registryPullQPS` / `registryBurst` (primary fix)

Set higher limits in the **kubelet configuration** (the CLI flags `--registry-qps` /
`--registry-burst` are deprecated; prefer the config file). Example kubelet config:

```yaml
# kubelet config (KubeletConfiguration)
registryPullQPS: 50
registryBurst: 100
serializeImagePulls: false   # allow parallel pulls (see below)
```

Or in the JSON form some distros use (`/etc/kubernetes/kubelet/kubelet-config.json`):

```json
{
  "registryPullQPS": 50,
  "registryBurst": 100,
  "serializeImagePulls": false
}
```

Setting `registryPullQPS: 0` disables the limit entirely, but that removes the safety valve
and can overwhelm a slow node or registry — prefer a concrete, tuned value.

The right value scales with **max pods per node**: a node that can hold 110 pods needs a
much higher pull QPS than one holding 8. A common heuristic is to size QPS/burst to your
`maxPods` so a cold node can start its full complement without stalling.

### 2. Enable parallel image pulls

By default (older Kubernetes) `serializeImagePulls: true` forces one pull at a time, which
makes the QPS ceiling bite harder. Setting `serializeImagePulls: false` lets the kubelet
pull multiple images concurrently, which shortens the burst window. You can bound the
concurrency with `maxParallelImagePulls`.

### 3. Applying it on managed node groups

You do not edit nodes by hand in most managed platforms — you bake the config into the node
bootstrap:

- **EKS (managed node groups / self-managed / Karpenter):** inject the settings via the
  node bootstrap. For AL2/AL2023 use the bootstrap `--kubelet-extra-args` or a MIME
  user-data block that patches `kubelet-config.json` (e.g. set `registryPullQPS`/
  `registryBurst`) before the kubelet starts. For Karpenter, set them in the `EC2NodeClass`
  user data / `kubelet` config.
- **AKS:** use a **custom kubelet configuration** on the node pool to set
  `registryPullQPS` and `registryBurst`.
- **GKE:** use a node system config to adjust kubelet parameters.

Roll the change out by cycling nodes (a new node group / new launch template), since kubelet
config is read at startup.

### 4. Reduce the pull pressure instead of raising limits

Sometimes the better fix is fewer/faster pulls:

- **Pre-pull / bake images into the AMI or node image** so cold nodes do not pull on
  startup.
- **Run a pull-through cache / mirror** (or a registry pull-through cache) close to the
  nodes to speed each pull and dodge upstream registry limits.
- **Slow the roll.** Lower surge / max-unavailable (or Karpenter consolidation
  aggressiveness) so fewer cold nodes come up simultaneously.
- **Shrink images** and reduce the number of distinct images per node.
- **Fix the node's disk I/O.** If pulls are slow because the root/EBS volume is
  throttled, faster/larger volumes (or more IOPS/throughput) reduce time-in-flight and the
  pile-up.

---

## Diagnosing a pull storm

Before (or after) changing limits, confirm the shape of the problem:

- **Count image-pull failures across the cluster** to see if they cluster on specific
  (usually new) nodes:

  ```bash
  kubectl get events -A --field-selector reason=Failed \
    -o wide | grep -i "pull QPS exceeded"
  ```

- **Find which node the failing pods landed on** — a pull storm concentrates on cold nodes:

  ```bash
  kubectl get pods -A -o wide | grep -Ei "ImagePullBackOff|ErrImagePull"
  ```

- **Measure how long pulls actually take.** If pulls are slow, the limiter is a symptom,
  not the cause. The kubelet exposes pull-duration metrics:

  ```
  kubelet_image_pull_duration_seconds        # histogram of pull time
  kubelet_running_pods                        # pods per node
  ```

  A high `kubelet_image_pull_duration_seconds` alongside `pull QPS exceeded` points at slow
  pulls (disk I/O, image size, registry latency), not the QPS ceiling.

- **Watch the backoff clear on its own.** Because the kubelet retries with backoff, note
  whether pods recover within a minute or two (transient burst) or stay stuck (real
  bottleneck). Only the stuck case needs more than a limit bump.

---

## Config values reference

Effective defaults depend on Kubernetes version and distro — always confirm with the
`configz` check above. Rough guide:

| Setting | Older default | Newer default (1.29+) | What to set it to |
|---|---|---|---|
| `registryPullQPS` | `5` | `20` | Size to `maxPods` (e.g. `50`–`100` for dense nodes) |
| `registryBurst` | `10` | `40` | ~2× `registryPullQPS` |
| `serializeImagePulls` | `true` | `false` | `false` to allow parallel pulls |
| `maxParallelImagePulls` | unset (unbounded when parallel) | unset | Bound concurrency if disk/registry is a concern |

---

## Example: EKS AL2023 node bootstrap (user data)

Patch the kubelet config before the kubelet starts, via a MIME multipart user data block on
the launch template / node group:

```yaml
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="BOUNDARY"

--BOUNDARY
Content-Type: application/node.eks.aws

apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  kubelet:
    config:
      registryPullQPS: 50
      registryBurst: 100
      serializeImagePulls: false
--BOUNDARY--
```

For Karpenter, put the same values under the `EC2NodeClass` (or `kubelet` block on older
Karpenter APIs) so newly provisioned nodes inherit them. Cycle nodes to apply — kubelet
config is read only at startup.

---

## Example: pre-pull images with a DaemonSet

If you cannot bake images into the node image, a "pre-puller" DaemonSet warms the cache on
every node so real pods find images already present. It pulls the image, then idles:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: image-prepuller
spec:
  selector:
    matchLabels: { app: image-prepuller }
  template:
    metadata:
      labels: { app: image-prepuller }
    spec:
      # Pull heavy/critical images here; the initContainer pulls, main container idles.
      initContainers:
        - name: prepull-app
          image: a.b.c/image:0.1
          command: ["sh", "-c", "echo pulled"]
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
          resources:
            requests: { cpu: 10m, memory: 16Mi }
            limits: { cpu: 10m, memory: 16Mi }
```

This spreads pulls over normal operation instead of concentrating them on a cold node
during a roll. Note it still consumes a pod slot per node, so scope it to the images that
actually cause the storm.

---

## When *not* to just crank the limit

`registryPullQPS`/`registryBurst` protect the node and the registry. Setting them very high
(or to `0`) on a node that is disk-I/O- or network-bound simply moves the failure: pulls
still can't complete fast enough, and you may now hammer the registry into its own `429`
rate limits. Tune the limiter to the node's real pull capacity and `maxPods`, and address
slow pulls (I/O, image size, registry latency) separately.

---

## TL;DR

- `pull QPS exceeded` is the **kubelet** throttling **image pulls**, not the API server and
  not the registry.
- Controlled by **`registryPullQPS`** (default often `5`) and **`registryBurst`** (default
  often `10`); newer Kubernetes uses `20`/`40`.
- It fires when **many distinct images are pulled on one node at once** — classically a
  **cold node during patching/recycling, scale-out, or failover**.
- Fix by raising `registryPullQPS`/`registryBurst` in the **kubelet config**, enabling
  `serializeImagePulls: false`, pre-pulling/baking images, mirroring the registry, and
  slowing the roll.
- If raising limits does not help, the real bottleneck is likely **slow pulls** (disk I/O,
  image size, or registry latency) — fix that instead.

## Sources

- Kubelet registry settings and new defaults (`serializeImagePulls` false, `registry-qps`
  20 / `registry-burst` 40): [kubernetes/kubernetes PR #114552](https://github.com/kubernetes/kubernetes/pull/114552)
- Default `registryPullQPS: 5` / `registryBurst: 10` and the `pull QPS exceeded` event:
  [Azure/AKS issue #3138](https://github.com/Azure/AKS/issues/3138)
- Parallel image pulls (`serializeImagePulls`, `maxParallelImagePulls`):
  [Kubernetes blog — speeding up Pod startup](https://v1-35.docs.kubernetes.io/blog/2023/05/15/speed-up-pod-startup/)
- Sizing registry QPS to `maxPods` in EKS:
  [awslabs/amazon-eks-ami PR #1801](https://github.com/awslabs/amazon-eks-ami/pull/1801)
- `pull QPS exceeded` correlated with node disk I/O limits:
  [IBM Cloud docs](https://cloud.ibm.com/docs/openshift?topic=openshift-ts-vpc-image-pull-qps)

Content was rephrased for compliance with licensing restrictions.
