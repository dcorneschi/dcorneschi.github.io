# Upgrading an EKS Control Plane from 1.34 to 1.35: What to Check

This is a pre-upgrade checklist for moving an Amazon EKS control plane from Kubernetes
`1.34` to `1.35`. The control plane upgrade itself is a single, mostly automated step,
but 1.35 removes and deprecates behavior that can break workloads and nodes if you
skip the prep work. Read this before you press the button.

> EKS only lets you upgrade one minor version at a time, and the control plane must be
> upgraded before the nodes. Once you move to 1.35 you cannot downgrade the control
> plane, so treat this as a one-way door and validate in a non-production cluster first.

---

## 1. The headline breaking change: cgroup v1 support is removed

Kubernetes 1.35 removes cgroup v1 support. On a node still using cgroup v1, the kubelet
refuses to start by default. This is the single most likely thing to take nodes offline
during the upgrade, so check it first.

What to verify per node type:

- **Amazon Linux 2023 (AL2023):** Uses cgroup v2 by default and matches upstream
  behavior. If you *manually* reconfigured AL2023 to use cgroup v1, you must either
  migrate the node to cgroup v2 or set `failCgroupV1: false` in the kubelet config.
- **Bottlerocket:** Bottlerocket 1.35 uses cgroup v2 by default but sets
  `failCgroupV1: false` in kubelet config for backward compatibility, so it keeps working.
- **Fargate:** Continues to use cgroup v1 (managed by AWS).
- **Ubuntu EKS AMI (Canonical-provided):** See the dedicated notes below — Ubuntu is a
  common EKS choice with its own considerations.
- **Custom AMIs / older Linux distros:** These are the real risk. Confirm the node OS
  supports and is running cgroup v2 before the nodes reach 1.35.

### Ubuntu EKS AMI considerations

Ubuntu EKS images are published and supported by **Canonical**, not AWS. That changes a
few things versus the AWS-optimized AL2023 and Bottlerocket images:

- **cgroup version:** Ubuntu 22.04 and later default to cgroup v2, so a current Ubuntu
  EKS AMI generally satisfies the 1.35 requirement. The risk cases are nodes still on
  **Ubuntu 20.04** (which defaults to cgroup v1) or any node where cgroup v1 was forced
  via the `systemd.unified_cgroup_hierarchy=0` kernel boot parameter. Confirm with
  `stat -fc %T /sys/fs/cgroup/` (expect `cgroup2fs`). If you must keep cgroup v1
  temporarily, set `failCgroupV1: false` in the kubelet config, but treat that as a
  stopgap, not a fix.
- **Image availability and support cadence:** Canonical builds Ubuntu EKS AMIs per
  Kubernetes minor version. Before upgrading the control plane, confirm a **1.35 Ubuntu
  AMI exists for your Ubuntu release and architecture** (amd64/arm64), and that it is on
  a supported Ubuntu LTS. Do not assume it lands the same day EKS adds 1.35 — Canonical
  releases on its own schedule.
- **containerd version:** Section 2 applies here too. Ubuntu 1.35 AMIs may ship
  containerd 1.x; check `containerd --version` on the node and plan the move to 2.0+
  before 1.36.
- **You own the bootstrap and kubelet config.** Because it is a self-managed AMI, the
  removed `--pod-infra-container-image` flag, removed/locked feature gates, and other
  changes in Section 3 are your responsibility to reconcile — AWS will not patch them
  for you as it does for AL2023/Bottlerocket.
- **Find the AMI:** query Canonical's published SSM parameters or the EKS AMI locator
  rather than hardcoding an AMI ID, e.g. search the SSM path
  `/aws/service/canonical/ubuntu/eks/...` for your Ubuntu release and Kubernetes 1.35.

Quick check on a node:

```bash
# "cgroup2fs" (magic 0x63677270) means cgroup v2 is active
stat -fc %T /sys/fs/cgroup/
# Expect: cgroup2fs
```

Note the control plane upgrade to 1.35 does not touch your nodes, but you should not
let 1.35 nodes join until they are confirmed on cgroup v2.

## 2. containerd 1.x is on its last supported release

Kubernetes 1.35 is the **last** release that supports containerd 1.x. You do not have
to switch for the 1.34 → 1.35 hop, but you must move to containerd 2.0+ before the
*next* upgrade (to 1.36). Plan it now so it is not a surprise later.

- EKS-optimized AMIs already ship a recent containerd, so managed node groups are
  generally fine.
- If you run custom AMIs, check the containerd version now.
- Watch the `kubelet_cri_losing_support` metric to find nodes on a containerd version
  that will soon be unsupported.

## 3. Removed and changed flags, APIs, and behaviors

These are the concrete removals and behavior changes in 1.35 that can break a node,
a manifest, a script, or a permission. Work through the ones that apply to you.

### Removed kubelet flags

- **`--pod-infra-container-image` (removed).** The kubelet no longer accepts this flag;
  the pause-image logic moved to the CRI. If your custom AMI or bootstrap script still
  passes it, the kubelet will fail to start. Remove it from your kubelet configuration
  and from `/var/lib/kubelet/kubeadm-flags.env` (kubeadm clusters) before nodes move to
  1.35. kubeadm attempts to strip it during upgrade, but non-kubeadm setups must remove
  it manually.

### Removed / locked feature gates

Feature gates that were removed or locked in 1.35 will cause a startup error if you
still set them explicitly on the API server, controller-manager, scheduler, or kubelet.
Remove these from your component flags and manifests:

- `SizeMemoryBackedVolumes` (removed — was GA)
- `ComponentSLIs` (removed — GA since 1.32)
- `UserNamespacesPodSecurityStandards` (removed)
- `ExecProbeTimeout` (locked to `true` — no longer settable)
- `AllowOverwriteTerminationGracePeriodSeconds` (locked)
- `DynamicResourceAllocation` (locked enabled — can no longer be disabled)
- kubeadm only: `RootlessControlPlane` and `ControlPlaneKubeletLocalMode` gates removed
  / graduated; User Namespaces replaces the former.
- Also note: `AllAlpha=true` no longer works on its own. Feature-gate dependencies are
  now validated at startup, so a gate cannot be enabled if a gate it depends on is
  disabled. Audit any component that sets `AllAlpha`/`AllBeta`.

### Removed environment variables

- `KUBECTL_OPENAPIV3_PATCH` (removed from kubectl)
- `KUBECTL_KYAML` still works, but `kubectl get -o kyaml` is now on by default; set
  `KUBECTL_KYAML=false` to opt out.

### Removed and dropped APIs

- **`StorageVersionMigration` `v1alpha1` (removed).** *Action required:* delete any
  `v1alpha1` StorageVersionMigration resources before upgrading. The `v1beta1` API
  replaces it.
- **`VolumeAttributesClass` in `storage.k8s.io/v1alpha1` (removed).** Use the GA
  `storage.k8s.io/v1` API (VAC went GA in 1.34).
- **kubectl dropped support for these older beta API versions** (the server may still
  serve them, but kubectl 1.35 will not send them). Bump your manifests to the stable
  versions:
  - `certificates.k8s.io/v1beta1` CertificateSigningRequest → `v1`
  - `discovery.k8s.io/v1beta1` EndpointSlice → `v1`
  - `networking.k8s.io/v1beta1` Ingress → `v1`
  - `policy/v1beta1` PodDisruptionBudget → `v1`

**How to check** (point kubectl at the cluster first with
`aws eks update-kubeconfig --name my-cluster --region us-west-2`):

*Removed `v1alpha1` resources — delete them before upgrading.* These live in the cluster,
so query the API server directly. If the group/version was never installed you get
`the server doesn't have a resource type ...`, which is a clean result:

```bash
# StorageVersionMigration v1alpha1 — any output here is action required
kubectl get storageversionmigrations.v1alpha1.storagemigration.k8s.io -A 2>/dev/null

# VolumeAttributesClass v1alpha1 (cluster-scoped)
kubectl get volumeattributesclasses.v1alpha1.storage.k8s.io 2>/dev/null

# Confirm which versions the server still serves for each group
kubectl get --raw /apis/storagemigration.k8s.io 2>/dev/null | jq '.versions'
kubectl get --raw /apis/storage.k8s.io | jq '.versions'
```

To delete a leftover `v1alpha1` StorageVersionMigration once you have confirmed it is done:

```bash
kubectl delete storageversionmigrations.v1alpha1.storagemigration.k8s.io <name>
```

For `VolumeAttributesClass`, recreate any objects under the GA `storage.k8s.io/v1` API
before removing the `v1alpha1` ones.

*Dropped beta apiVersions — bump manifests to stable.* The highest-value check is your
**source manifests and Helm charts**, because a live-cluster query returns the server's
normalized (current) version and hides what the object was originally applied with. Grep
your IaC repos for the four dropped versions:

```bash
# Search source manifests / Helm templates for the dropped beta apiVersions
grep -rEn 'apiVersion:\s*(certificates\.k8s\.io/v1beta1|discovery\.k8s\.io/v1beta1|networking\.k8s\.io/v1beta1|policy/v1beta1)' ./manifests ./charts
```

Then confirm nothing in the live cluster or Helm releases still ships them, using a
schema/ruleset scanner (same tools as Section 5):

```bash
# Live cluster + Helm releases, targeting the version you are moving to
pluto detect-all-in-cluster --target-versions k8s=v1.35
kubectl deprecations --k8s-version=v1.35.0        # kubepug via krew

# Rendered Helm output (catches versions that only exist after templating)
helm template -f values.yaml ./chart | kubepug --k8s-version=v1.35.0 --input-file=-
```

Note these four are *dropped by the kubectl client*, not necessarily removed from the API
server yet, so scanners keyed only to server-side removals may not flag them — the source
`grep` is the reliable check. See Section 5 for the full scanner comparison and caveats.

### Behavior changes that need action

- **`pods/exec`, `pods/attach`, `pods/portforward` now require `create` for WebSocket
  requests too.** Previously WebSocket requests only needed `get`. *Action required:*
  before upgrading, make sure any custom ClusterRoles/Roles granting these subresources
  include the `create` verb, or `kubectl exec`/`attach`/`port-forward` over WebSocket
  will be denied. Gated by `AuthorizePodWebsocketUpgradeCreatePermission` (on by default).
- **`kubectl exec` syntax is stricter.** `kubectl exec POD COMMAND` is no longer
  accepted; you must use `kubectl exec POD -- COMMAND`. Update scripts and automation.
- **cgroup v1 validation is now an error, not a warning** (see Section 1), driven by
  `failCgroupV1` defaulting to `true`.

## 4. Deprecations that do not block 1.35 but need a plan

These still work in 1.35 but are on a countdown. Log them in your upgrade tracker.

- **kube-proxy IPVS mode is deprecated** and slated for removal in 1.36. If your nodes
  run kube-proxy in `ipvs` mode, plan a migration. On Linux the recommended mode is
  `nftables`. How to check which mode you are running (point kubectl at the cluster
  first with `aws eks update-kubeconfig --name my-cluster --region us-west-2`):

  ```bash
  # 1. Read the mode straight from the kube-proxy ConfigMap.
  #    An empty value ("") means the default (iptables on Linux), not ipvs.
  kubectl -n kube-system get configmap kube-proxy \
    -o jsonpath='{.data.config\.conf}' | grep -E '^\s*mode:'
  # Look for: mode: "ipvs"  -> action needed

  # 2. Confirm what the running pods actually use (the flag/arg they booted with),
  #    in case the ConfigMap and the live DaemonSet have drifted.
  kubectl -n kube-system get ds kube-proxy \
    -o jsonpath='{.spec.template.spec.containers[0].args}'; echo
  kubectl -n kube-system get ds kube-proxy \
    -o jsonpath='{.spec.template.spec.containers[0].command}'; echo

  # 3. Ask a running kube-proxy pod which mode it selected (most authoritative).
  #    "Using ipvs Proxier" in the logs confirms ipvs is active.
  POD=$(kubectl -n kube-system get pods -l k8s-app=kube-proxy \
    -o jsonpath='{.items[0].metadata.name}')
  kubectl -n kube-system logs "$POD" | grep -iE 'using .* proxier|proxy mode'

  # 4. On a node, the presence of IPVS virtual servers is a direct signal.
  #    Empty output means ipvs is not in use on that node.
  sudo ipvsadm -Ln 2>/dev/null | head
  ```

  If any of these show `ipvs`, log the migration to `nftables` (or `iptables`) as a
  1.36 follow-up. It still works in 1.35, so this is a plan-ahead item, not a blocker.
- **Ingress NGINX retirement (upstream, ~March 2026):** The upstream project is retiring
  Ingress NGINX, meaning no more bug fixes or security patches. This is not tied to the
  1.35 API but it lands in the same time window. If you depend on it, start evaluating
  Gateway API or a third-party ingress controller. None of the alternatives are drop-in
  replacements, so budget engineering time.

## 5. Standard pre-upgrade hygiene (do this every upgrade)

Independent of what changed in 1.35, run these checks:

- **Scan for deprecated/removed API usage.** See the section below for the exact
  commands. There is no built-in `kubectl deprecations` subcommand, so use the API
  server's dry-run warnings or a dedicated scanner.
- **Check control-plane-to-node version skew.** Kubernetes supports nodes up to a few
  minor versions behind the control plane, but do not let the gap widen. After the
  control plane hits 1.35, upgrade nodes promptly.
- **Update EKS add-ons.** Align managed add-ons (VPC CNI, CoreDNS, kube-proxy, EBS/EFS
  CSI drivers) with versions compatible with 1.35. kube-proxy in particular should track
  the control plane version.
- **Review self-managed CSI sidecars.** If you self-manage CSI sidecar containers rather
  than using AWS-managed ones, confirm they are compatible with 1.35.
- **Confirm IAM, PodIdentity/IRSA, and admission webhooks** still work against the new
  API server. Broken webhooks can block all writes to the cluster.
- **Check quotas and capacity** so new nodes can be provisioned during the rolling node
  replacement.

### Start with EKS Cluster Insights (AWS-native check)

Before reaching for third-party scanners, check **EKS Cluster Insights**. It is a
built-in, no-cost feature that automatically and continuously scans your cluster against
an AWS-curated list of known upgrade-impacting issues, then recommends fixes with links
to the relevant AWS and Kubernetes docs. AWS updates the check list as each new
Kubernetes version ships, so it reflects the actual 1.35 changes.

What it covers, by insight category:

- **Upgrade insights:** Flag things that would block or break a version upgrade, such as
  deprecated/removed API usage and add-on compatibility problems. This is the one to
  read before a 1.34 → 1.35 upgrade.
- **Configuration insights:** Detect misconfigurations in EKS Hybrid Nodes setups (for
  example, broken control-plane-to-webhook communication or `kubectl exec`/`logs`).
- **Rollback readiness insights:** After an upgrade, flag issues that would prevent
  rolling back. These are point-in-time checks and are only generated for clusters
  upgraded within the last 7 days.

How it works and how to use it:

- Insights refresh automatically every 24 hours; you can also refresh on demand after
  fixing an issue to confirm it cleared.
- EKS creates a cluster access entry automatically so it can read the cluster to
  generate insights. No setup required.
- View them in the **Upgrade insights** tab of the EKS console observability dashboard,
  or via the CLI:

```bash
# List all insights for a cluster
aws eks list-insights --cluster-name my-cluster --region us-west-2

# Filter to upgrade readiness, targeting the version you are moving to
aws eks list-insights --cluster-name my-cluster --region us-west-2 \
  --filter '{"categories":["UPGRADE_READINESS"],"kubernetesVersions":["1.35"]}'

# Get details and remediation for a specific finding
aws eks describe-insight --cluster-name my-cluster --region us-west-2 --id <insight-id>
```

Resolve any upgrade insights before you update the control plane. Note that Cluster
Insights is EKS's own scan; still run a manifest/Helm scanner (below) for coverage of
resources that only live in your source repos, not the live cluster.

### How to scan for deprecated APIs

`kubectl` does **not** ship a built-in deprecation scanner, so a bare
`kubectl deprecations` fails. You can add one as a krew plugin (Option D below), or use
one of the other options here.

**Option A — API server dry-run warnings (built in, nothing to install).** The API
server emits deprecation warnings on requests, so a server-side dry-run surfaces
deprecated APIs in what you are applying:

```bash
# Warnings print for deprecated APIs in the manifest
kubectl apply --dry-run=server -f manifest.yaml

# Turn deprecation warnings into a hard failure (useful in CI)
kubectl apply --dry-run=server --warnings-as-errors -f manifest.yaml
```

This only checks what you apply, not everything already stored in the cluster.

**Option B — `kubent` (kube-no-trouble): scans the live cluster, Helm releases, and files.**

```bash
brew install kubent   # or: sh -c "$(curl -sSL https://git.io/install-kubent)"

# Scan the current cluster context, targeting the version you are moving to
kubent -t 1.35
```

Interpreting the output:

- `kubent` only prints a findings table when it detects something. A run that loads its
  rulesets, reports the resource counts (e.g. `Retrieved 34 resources from collector`),
  and then **stops with no table means nothing deprecated/removed was found** in what it
  scanned. Confirm with the exit code — `echo $?` right after; by default `kubent` exits
  non-zero only when it finds deprecated APIs, so `0` is a clean scan.
- Check which rulesets loaded. An older build may only carry rulesets up to, say,
  `deprecated-1-32` plus `deprecated-future`, with no `deprecated-1-35` ruleset. In that
  case a clean result means "nothing removed in the versions it knows about" — still
  cross-check the removed APIs in Section 3 (for example `StorageVersionMigration` and
  `VolumeAttributesClass` `v1alpha1`) by hand, and update `kubent` to the newest release.
- `Retrieved 0 resources from collector name="Helm v3"` just means no Helm 3 releases
  were found; it is not an error.
- A clean `kubent` run only clears the **API-version** dimension. It never sees the
  node/config-level 1.35 changes (cgroup v1, the removed kubelet flag, feature gates,
  the `pods/exec` RBAC change), so Sections 1 and 3 still apply.

**Option C — `pluto`: good for manifests, Helm charts, and live-cluster scans.**

```bash
brew install FairwindsOps/tap/pluto

# Scan a directory of manifests (your Git/IaC repos)
pluto detect-files -d ./manifests --target-versions k8s=v1.35

# Scan live Helm releases in the cluster only
pluto detect-helm --target-versions k8s=v1.35

# Combined in-cluster scan: live Helm releases AND live API resources in one pass
pluto detect-all-in-cluster --target-versions k8s=v1.35
```

Notes on using `pluto`:

- `detect-all-in-cluster` is the best single command for the live-cluster check because
  it covers both Helm releases and raw API resources at once. It reads the cluster
  through your current kubeconfig context, so point that at the right cluster first.
- `--target-versions k8s=v1.35` sets the version you are upgrading *to*.
- **Live scans do not see your Git repos.** `detect-all-in-cluster` only inspects what is
  already running. Deprecated `apiVersion`s that live only in unapplied manifests are
  invisible to it, so also run `detect-files -d ./manifests` against your source.
- **API-server normalization caveat.** If you query the API server for an object, it
  converts and returns the *current* `apiVersion`, hiding the version the object was
  originally applied with. Pluto works around this for Helm (it reads the stored release
  manifest), but pure live API-resource detection can still miss objects whose version
  was rewritten on read. This is another reason to scan your source manifests too.

**Option D — `kubectl deprecations` (the `deprecations` krew plugin / kubepug).**
There *is* a `kubectl deprecations` command, but only after you install the `deprecations`
krew plugin, which is [kubepug](https://github.com/kubepug/kubepug) ("Kubernetes
PreUpGrade Checker"):

```bash
kubectl krew install deprecations

# Check the live cluster against the target version (note the leading v and full patch)
kubectl deprecations --k8s-version=v1.35.0

# Check rendered manifests / Helm output instead of the live cluster
helm template -f values.yaml ./chart | kubepug --k8s-version=v1.35.0 --input-file=-
```

Why it is worth having alongside `kubent`/`pluto`:

- **Schema-based, not ruleset-based.** kubepug checks against the target version's
  Kubernetes OpenAPI/swagger schema rather than hand-maintained rule files, so it is not
  limited by whichever `deprecated-1-XX` rulesets a given scanner build happens to ship.
  That makes it well suited to a fresh release like 1.35.
- Same **api-server normalization** caveat as any live scan: objects are reported at the
  server's converted version, so also run it against your source manifests (the
  `--input-file=-` form above).
- Still only the **API-version** dimension — it will not catch the node/config-level
  1.35 changes in Sections 1 and 3.

**Option E — `kubectl get --raw /apis` (inventory only, NOT a deprecation scanner).**
Useful for a quick look at what API versions the control plane advertises, but do not
mistake it for a readiness check:

```bash
# Preferred (storage) version per API group, e.g. apps/v1, networking.k8s.io/v1
kubectl get --raw /apis | jq -r '.groups[].preferredVersion.groupVersion'

# Full list of served versions for one group (preferred + still-served older ones)
kubectl get --raw /apis/networking.k8s.io | jq '.versions'
```

Why this is not enough on its own:

- It reports the version the server **prefers today**, not the version your objects were
  **created with**. An object applied as `networking.k8s.io/v1beta1` still shows the
  group's preferred version as `v1` here — the same api-server normalization trap noted
  under `pluto` above.
- The first command shows only the *preferred* version, not every served version, so a
  still-served deprecated version can be missed. Query a group directly (second command)
  to see its full `versions` list.

Treat this as an inventory aid. To actually find deprecated `apiVersion`s in your
workloads, use Options A–D (a source-manifest scan plus a live scan).

> These scanners only find **removed or deprecated API versions** (for example, an old
> `apiVersion` you need to bump). They do **not** catch the highest-risk 1.35 changes,
> which are node- and runtime-level: cgroup v1 removal (Section 1), the removed
> `--pod-infra-container-image` kubelet flag (Section 3), and containerd 1.x reaching its
> last supported release (Section 2). Run a scanner *and* work through those sections.

## 6. New capabilities in 1.35 worth knowing about

Not upgrade blockers, but things you may want to adopt after the move:

- **In-Place Pod Resize (GA):** Adjust a Pod's CPU/memory without restarting it. Good
  for stateful and batch workloads that previously needed a full recreate to resize.
- **`PreferSameNode` traffic distribution (GA):** A new `trafficDistribution` option for
  Services that strictly prefers endpoints on the local node, falling back to remote.
- **StatefulSet `maxUnavailable` (Beta):** Update multiple StatefulSet Pods in parallel
  (e.g. `maxUnavailable: 3` or `10%`) to shrink maintenance windows.
- **Windows Server 2025 support** is added in EKS 1.35.

## 7. Suggested upgrade sequence

1. Read the [EKS standard support release notes](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions-standard.html)
   and the upstream 1.35 changelog end to end.
2. Run API deprecation scans and fix flagged manifests.
3. Verify all nodes are on cgroup v2 (Section 1) and custom AMIs drop the removed
   kubelet flag (Section 3).
4. Confirm add-on and CSI compatibility, and update add-ons as needed.
5. Upgrade the **control plane** to 1.35 (single minor-version step).
6. Validate the API server, webhooks, and workloads on a test cluster.
7. Upgrade **node groups** to 1.35, watching for any nodes that fail to start.
8. Log the containerd 2.0 migration and IPVS deprecation as follow-ups for the 1.36 hop.

---

### Sources

- [Review release notes for Kubernetes versions on standard support (Amazon EKS)](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions-standard.html)
- [Kubernetes v1.35 Sneak Peek (Kubernetes Blog)](https://kubernetes.io/blog/2025/11/26/kubernetes-v1-35-sneak-peek/)
- [Kubernetes v1.35 release announcement](https://kubernetes.io/blog/2025/12/17/kubernetes-v1-35-release/)
- [Understand the Kubernetes version lifecycle on EKS](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
- [Kubernetes v1.35.0 changelog (Deprecation, API Change, and Urgent Upgrade Notes sections)](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.35.md)
- [Prepare for Kubernetes version upgrades with cluster insights (Amazon EKS)](https://docs.aws.amazon.com/eks/latest/userguide/cluster-insights.html)
- [kubepug — Kubernetes PreUpGrade Checker (the `deprecations` krew plugin)](https://github.com/kubepug/kubepug)
