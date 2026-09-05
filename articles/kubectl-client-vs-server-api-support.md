# Checking Which APIs the Client and Server Support (and the Differences)

When you plan a Kubernetes upgrade or debug a version-skew problem, a natural question is:
*what APIs does my `kubectl` support, what does the API server support, and where do they
disagree?* This article shows how to enumerate both sides and how to get at the
differences.

There is an important asymmetry to understand up front:

- **The server exposes its full API surface through discovery.** You can list exactly
  which groups, versions, and resources it serves, over the wire, at any time.
- **The client does not expose a comparable list.** `kubectl`'s knowledge of API versions
  is compiled into the binary. There is no flag that prints "every apiVersion I would
  send." You infer the client's support from its version and behavior.

So you can enumerate the server precisely, but "what the client supports" is inferred —
and there is **no single built-in command that prints a client-vs-server diff**. Below is
how to get as close as possible.

---

## What the server supports

These commands query the server's discovery endpoints. They report the **server**,
regardless of your `kubectl` version.

```bash
# Every API resource plus its apiVersion the server serves right now
kubectl api-resources -o wide

# Every group/version the server serves (authoritative server discovery)
kubectl api-versions

# Preferred (storage) version per group — what the server defaults to
kubectl get --raw /apis | jq -r '.groups[].preferredVersion.groupVersion'

# Full list of served versions for one group (preferred + still-served older ones)
kubectl get --raw /apis/networking.k8s.io | jq '.versions'
```

`kubectl api-resources` and `kubectl api-versions` are retrieved from the server's
discovery API (`GET /api`, `GET /apis`). Even an old client reports the current server
here, because the data comes from the server, not the client.

Sample output — `kubectl api-versions` (truncated) lists one line per served
group/version:

```text
admissionregistration.k8s.io/v1
apiextensions.k8s.io/v1
apps/v1
autoscaling/v2
batch/v1
networking.k8s.io/v1
policy/v1
storage.k8s.io/v1
v1
```

Sample output — `kubectl api-resources -o wide` (truncated) shows the resource, its
apiVersion (the `APIVERSION` column), namespaced flag, kind, and verbs:

```text
NAME         SHORTNAMES   APIVERSION           NAMESPACED   KIND        VERBS
pods         po           v1                   true         Pod         [create delete get list patch update watch]
deployments  deploy       apps/v1              true         Deployment  [create delete get list patch update watch]
ingresses    ing          networking.k8s.io/v1 true         Ingress     [create delete get list patch update watch]
```

Sample output — the preferred-version query returns just the storage version per group:

```text
apps/v1
networking.k8s.io/v1
policy/v1
storage.k8s.io/v1
```

> Caveat: `preferredVersion` shows the version the server prefers **today**, not every
> version it still serves. A deprecated version can still be served while no longer being
> preferred. Query the group directly (the fourth command) to see its full `versions`
> list.

## What the client "supports"

There is no flag that dumps the client's known apiVersions. What you can check is the
client version and the skew against the server:

```bash
# Client and server versions side by side (also warns on unsupported skew)
kubectl version

# Structured, for scripting
kubectl version -o json \
  | jq '{client: .clientVersion.gitVersion, server: .serverVersion.gitVersion}'
```

Sample output — `kubectl version` prints both sides and warns when the skew is
unsupported:

```text
Client Version: v1.32.2
Kustomize Version: v5.5.0
Server Version: v1.35.0-eks-abc1234
WARNING: version difference between client (1.32) and server (1.35) exceeds the supported minor version skew of +/-1
```

The `-eks-abc1234` suffix on the server version is EKS's build identifier — the upstream
Kubernetes minor (`1.35`) is what matters for API support.

The client's supported API set is effectively "the APIs that shipped in that `kubectl`
minor version." A 1.32 client knows 1.32-era APIs: it will not **send** versions dropped
in a later client (for example the `v1beta1` API versions that kubectl 1.35 stopped
sending), and it will not **understand** resource types added after 1.32.

### Probing whether the client knows a specific type/version

`kubectl explain` is the closest thing to interrogating the client's schema knowledge. It
resolves a resource's schema (from the server's OpenAPI, negotiated by the client), and
its `--api-version` flag lets you ask about one specific group/version:

```bash
# Explain a resource at a specific apiVersion — errors if that GV isn't available
kubectl explain ingress --api-version=networking.k8s.io/v1
kubectl explain ingress --api-version=networking.k8s.io/v1beta1   # fails once dropped

# The VERSION line tells you which group/version resolved
kubectl explain ingress | grep -E '^(KIND|VERSION):'
```

If `--api-version` names a group/version the pair (client + server) can't resolve, you get
an error like `couldn't find resource for "networking.k8s.io/v1beta1, Kind=Ingress"` —
a quick way to confirm a version is gone.

### Migrating manifests between apiVersions with kubectl-convert

`kubectl convert` was split out of the main binary; it's now the separate **`kubectl-convert`**
plugin. It rewrites a manifest from one apiVersion to another, handling the field/shape
changes that a plain string swap misses:

```bash
# Install (Krew) or grab the released binary
kubectl krew install convert          # or download the kubectl-convert binary

# Convert a manifest to a target apiVersion
kubectl convert -f old-ingress.yaml --output-version networking.k8s.io/v1

# Convert to whatever the client considers the latest for that kind
kubectl convert -f old-ingress.yaml
```

Note `kubectl-convert` uses the **client's** compiled-in knowledge, so it can only convert
to versions that client understands — another reason to run a client matching your target
version.

### Aggregated discovery: how modern clients enumerate the server in one shot

Older clients discovered APIs by hitting every group/version endpoint — a request storm.
Kubernetes added **aggregated discovery** (KEP-3352; beta and on by default from ~1.27,
GA later), which lets a client fetch the whole API surface in one request per root (`/api`
and `/apis`) by sending an `Accept` header for the `APIGroupDiscoveryList` type. You can
request it directly:

```bash
# Aggregated discovery document for all named groups, in one request
kubectl get --raw /apis \
  -H 'Accept: application/json;g=apidiscovery.k8s.io;v=v2;as=APIGroupDiscoveryList' \
  | jq -r '.items[] | .metadata.name as $g
           | .versions[] | "\($g)/\(.version) (freshness=\(.freshness))"'

# Older beta variant if v2 isn't served yet
kubectl get --raw /apis \
  -H 'Accept: application/json;g=apidiscovery.k8s.io;v=v2beta1;as=APIGroupDiscoveryList'
```

The aggregated document also reports a per-version **`freshness`** field (`Current` vs
`Stale`), which matters in mixed-version control planes (see the EKS note below). This is
purely a server-side enumeration — it tells you what the *server* serves, efficiently; it
still doesn't enumerate the client's compiled-in set.

## Getting at the differences

No single command prints a client-vs-server diff. The practical approaches:

### 1. Check the skew directly (usually what actually matters)

```bash
kubectl version -o json \
  | jq -r '"client=\(.clientVersion.gitVersion) server=\(.serverVersion.gitVersion)"'
```

Kubernetes supports `kubectl` within **one minor version** of the API server (n-1, n,
n+1). If client and server are more than one minor apart, you are outside supported skew
and the client may mishandle newer server APIs.

### 2. Diff server discovery between two clusters

To see which APIs two **servers** differ on (for example an old cluster versus a new 1.35
cluster), diff their discovery output:

```bash
kubectl --context old-cluster api-versions | sort > /tmp/old.txt
kubectl --context new-cluster api-versions | sort > /tmp/new.txt
diff /tmp/old.txt /tmp/new.txt
```

### 3. Probe whether the server still serves a specific version

```bash
# Does the server still serve a deprecated beta version?
kubectl get --raw /apis/networking.k8s.io | jq -r '.versions[].groupVersion'
# Look for networking.k8s.io/v1beta1 in the output
```

### 4. Compare two client binaries against the same server

This surfaces behavioral differences between client versions — types a newer client knows
and an older one drops or renders differently:

```bash
kubectl-1.32 api-resources -o wide | sort > /tmp/c132.txt
kubectl-1.35 api-resources -o wide | sort > /tmp/c135.txt
diff /tmp/c132.txt /tmp/c135.txt
```

## The api-server normalization trap

All live-cluster checks share one caveat: when you read an object back, the API server
returns it at its **current, normalized** version, hiding the `apiVersion` it was
originally created with. An object applied as `networking.k8s.io/v1beta1` is reported as
`networking.k8s.io/v1` on read.

That means discovery and `kubectl get -o yaml` cannot reliably tell you which deprecated
versions your manifests still use. To find those, scan your **source manifests and Helm
charts**, not just the live cluster:

```bash
grep -rEn 'apiVersion:\s*(networking\.k8s\.io/v1beta1|policy/v1beta1)' ./manifests ./charts
```

## For upgrade readiness, use a target-version scanner

Do not try to hand-diff APIs to decide whether you are clear to upgrade. Purpose-built
tools check your workloads against the version you are moving **to**:

```bash
# Schema-based: checks against the target version's OpenAPI schema
kubectl deprecations --k8s-version=v1.35.0        # kubepug via the deprecations krew plugin

# Ruleset/live-cluster scans
pluto detect-all-in-cluster --target-versions k8s=v1.35
kubent -t 1.35

# Server-driven: the API server emits deprecation warnings on the request itself
kubectl apply --dry-run=server -f manifest.yaml
kubectl apply --dry-run=server --warnings-as-errors -f manifest.yaml   # fail in CI
```

`kubepug` is schema-based (it reads the target version's Kubernetes OpenAPI schema), so
it is not limited to whatever rulesets a given scanner build ships — a good fit for a
brand-new release. `pluto` and `kubent` are ruleset-based and also scan Helm releases and
manifests. The `--dry-run=server` approach needs nothing installed but only checks what
you apply, not everything already stored in the cluster.

## Note on `kubectl` and cluster upgrades

`kubectl` is a standalone client binary. It is not tied to the cluster upgrade and can be
updated before, during, or after a control plane upgrade. On EKS, `aws eks
update-kubeconfig` only rewrites your kubeconfig (endpoint and auth) — it does **not**
change the `kubectl` binary; you update that separately (Homebrew, the published binary,
`asdf`, etc.).

For clean results, keep `kubectl` within one minor of the API server, and prefer matching
it to the **target** version before you validate an upgrade. A stale client will not warn
you about APIs the newer version deprecates or removes, so its silence is not evidence
that you are clear.

## On Amazon EKS

The distinction between "server" and "client" is especially clean on EKS, because AWS runs
the control plane and you run `kubectl` wherever you like.

### Check the control plane (server) version without kubectl

You can read the control plane's Kubernetes version straight from the EKS API — no cluster
auth or `kubectl` needed:

```bash
# The control plane's Kubernetes minor version
aws eks describe-cluster --name my-cluster --region us-west-2 \
  --query 'cluster.version' --output text
# e.g. 1.35

# Platform version (EKS's own patch/revision on top of the k8s minor) and status
aws eks describe-cluster --name my-cluster --region us-west-2 \
  --query 'cluster.{k8s:version,platform:platformVersion,status:status}'
```

`cluster.version` is the upstream Kubernetes minor that governs which APIs the server
serves. `platformVersion` is EKS-specific (security/feature revisions within that minor)
and does not change the Kubernetes API surface.

Then point `kubectl` at it and query discovery as usual — note that
`aws eks update-kubeconfig` only rewrites kubeconfig (endpoint + auth); it does **not**
install or change your `kubectl` binary:

```bash
aws eks update-kubeconfig --name my-cluster --region us-west-2
kubectl api-versions          # server discovery, as above
kubectl version               # confirm client-vs-server skew
```

### EKS control plane is multi-instance — mind discovery freshness

An EKS control plane runs multiple API server instances behind an endpoint. **During an
EKS control plane upgrade** the instances are replaced in a rolling fashion, so for a short
window requests can hit servers on different minor versions. This is exactly the
mixed-version scenario aggregated discovery's `freshness` field flags as `Stale`. If
discovery output looks inconsistent mid-upgrade, re-check after the update completes
(`aws eks describe-cluster ... --query 'cluster.status'` returns `ACTIVE`).

### Detecting deprecated-API usage on EKS

Discovery and the normalization trap above mean live queries won't reliably tell you which
deprecated versions your workloads still call. On EKS, the AWS-native way to find that is
**EKS Cluster Insights**, which detects deprecated/removed API usage from the last 30 days
of real API server traffic and reports the offending client. See the companion doc
`eks-cluster-insights.md` for the full data model and commands. In short:

```bash
aws eks list-insights --region us-west-2 --cluster-name my-cluster \
  --filter '{"categories":["UPGRADE_READINESS"],"kubernetesVersions":["1.35"]}'
```

Use Insights (live traffic) *and* a source-manifest scanner (`kubent`/`pluto`/`kubepug`)
*and* the version-specific node/runtime checklist — no single one of them is complete.

## Quick reference

| Goal | Command |
|------|---------|
| List server API resources | `kubectl api-resources -o wide` |
| List server group/versions | `kubectl api-versions` |
| Preferred version per group | `kubectl get --raw /apis \| jq -r '.groups[].preferredVersion.groupVersion'` |
| All served versions for a group | `kubectl get --raw /apis/<group> \| jq '.versions'` |
| Client vs server version + skew | `kubectl version` |
| Probe a specific type/version (client) | `kubectl explain <res> --api-version=<group/version>` |
| Convert a manifest between apiVersions | `kubectl convert -f f.yaml --output-version <group/version>` |
| Aggregated discovery (one request) | `kubectl get --raw /apis -H 'Accept: application/json;g=apidiscovery.k8s.io;v=v2;as=APIGroupDiscoveryList'` |
| EKS control plane version (no kubectl) | `aws eks describe-cluster --name <c> --query 'cluster.version'` |
| EKS deprecated-API usage (live traffic) | `aws eks list-insights --cluster-name <c> --region <r>` |
| Diff two servers' APIs | `diff <(kubectl --context a api-versions \| sort) <(kubectl --context b api-versions \| sort)` |
| Find deprecated versions in source | `grep -rEn 'apiVersion:.*v1beta1' ./manifests` |
| Upgrade readiness scan | `kubectl deprecations --k8s-version=vX.Y.0` |

---

### Sources

- [Kubernetes version skew policy](https://kubernetes.io/releases/version-skew-policy/)
- [Kubernetes API and Feature deprecation policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)
- [Deprecated API migration guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)
- [The Kubernetes API — Discovery API and aggregated discovery](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)
- [KEP-3352: Aggregated Discovery](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/3352-aggregated-discovery/README.md)
- [Mixed Version Proxy (discovery freshness in multi-version control planes)](https://kubernetes.io/docs/concepts/architecture/mixed-version-proxy/)
- [Amazon EKS: DescribeCluster API](https://docs.aws.amazon.com/eks/latest/APIReference/API_DescribeCluster.html)
- [Prepare for Kubernetes version upgrades with cluster insights (Amazon EKS)](https://docs.aws.amazon.com/eks/latest/userguide/cluster-insights.html)
- [kubepug — Kubernetes PreUpGrade Checker (the `deprecations` krew plugin)](https://github.com/kubepug/kubepug)
