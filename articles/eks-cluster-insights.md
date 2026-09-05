# Amazon EKS Cluster Insights: What It Verifies and How It Works

Amazon EKS Cluster Insights is a built-in, no-cost feature that continuously scans every
EKS cluster against an AWS-curated list of known issues, then reports findings with
remediation advice and links to the relevant AWS and Kubernetes docs. Its headline job is
**upgrade readiness** — most notably catching **deprecated and removed Kubernetes API
usage** before you move the control plane to a new minor version — but it now covers more
than that.

This article explains exactly what Insights checks, the categories it reports, the data
model behind a finding (including every field in the deprecated-API detail), how the
detection actually works, the CLI/API surface, its permissions model, and the important
gaps you still have to cover with other tools.

> If you are preparing a specific upgrade, pair this with the version-specific checklist.
> Insights tells you *what the live cluster is doing*; the changelog and node/runtime
> checks tell you the rest. See the "What Insights does NOT catch" section near the end.

---

## 1. The three insight categories

Every EKS cluster is scanned automatically and recurringly. Findings fall into three
categories:

### Upgrade insights (UPGRADE_READINESS)

The core category. These identify anything that could block or break a Kubernetes
**version upgrade**, and run on every EKS cluster continuously. AWS updates the check list
with each new Kubernetes release, so the checks reflect the actual changes in the version
you're targeting. This category is where **deprecated/removed API detection** and
**add-on compatibility** live.

### Configuration insights

These scan clusters with **EKS Hybrid Nodes** for misconfigurations that impair cluster
or workload functionality — for example, broken Kubernetes control-plane-to-webhook
communication, or `kubectl exec` / `kubectl logs` not working. They surface the issue and
give remediation steps to get a hybrid-nodes setup fully functional.

### Rollback readiness insights (ROLLBACK_READINESS)

These identify issues that would prevent **rolling back** to a previous Kubernetes version
after an upgrade. Key characteristics:

- **Only generated for clusters upgraded within the last 7 days.** After the 7-day
  rollback-eligibility window expires, they are no longer generated.
- **Point-in-time, not continuous.** They reflect the cluster state at the moment of
  evaluation, unlike upgrade insights which run continuously.
- They check **API usage compatibility** (incompatibilities during API version
  graduation — a previous API version being removed, new resources that don't exist in
  the desired version, and new field or enum changes), **cluster health**, **kubelet and
  kube-proxy version skew**, **EKS managed add-on compatibility**, and, for **Auto Mode**
  clusters, **disruption budget and annotation** checks.
- **Managed add-ons only:** rollback insights only check EKS-managed add-on versions. If
  you self-manage an add-on, or override a managed add-on's version outside the EKS
  add-on lifecycle, Insights will not detect its incompatibilities — that's on you.

---

## 2. Insight status values (and what blocks an upgrade)

Every insight — and every affected resource within it — carries an `insightStatus` with
one of four `status` values plus a human-readable `reason`:

| Status | Meaning |
|---|---|
| `PASSING` | The check found no problem (e.g. "No deprecated API usage detected within the last 30 days."). |
| `WARNING` | Advisory. Something to plan for, but does not block the operation. |
| `ERROR` | Action required. The issue must be resolved before the operation. |
| `UNKNOWN` | EKS could not determine the result. Treat like an error until cleared. |

**For rollbacks:** insights with `ERROR` or `UNKNOWN` block the rollback until resolved;
`WARNING` is advisory only. You can override with the `--force` flag to proceed at your
own risk.

**For upgrades:** AWS had rolled out (and then **temporarily rolled back**) a feature that
would require a `--force` flag to upgrade when certain insight issues existed. As of this
writing that enforcement is disabled, so `ERROR` insights don't hard-block the control
plane update via the API — but you should still treat an `ERROR` as a must-fix before
upgrading. Don't rely on the tooling to stop you.

---

## 3. What a finding actually contains (the data model)

Understanding the `DescribeInsight` / `describe-insight` response tells you exactly what
Insights knows. The `insight` object includes:

- **`id`** – the insight's unique identifier.
- **`name`** – e.g. `Deprecated APIs removed in Kubernetes v1.29`.
- **`category`** – `UPGRADE_READINESS`, `CONFIGURATION`, or `ROLLBACK_READINESS`.
- **`kubernetesVersion`** – the target version the check is evaluated against.
- **`description`** – what the check looks for and why it matters.
- **`recommendation`** – the remediation action to take.
- **`insightStatus`** – `{ status, reason }` as above.
- **`lastRefreshTime`** – when the check last ran.
- **`lastTransitionTime`** – when the status last changed.
- **`additionalInfo`** – a map of extra links/context (docs pointers).
- **`resources[]`** – the specific affected resources, each with:
  - `arn` (if applicable),
  - `insightStatus` (`{ status, reason }`) for that individual resource,
  - **`kubernetesResourceUri`** — e.g. `/apis/policy/v1beta1/podsecuritypolicies/null`,
    which pinpoints the offending API path.
- **`categorySpecificSummary`** – the interesting part, with two sub-structures:
  - **`addonCompatibilityDetails[]`** – `{ name, compatibleVersions[] }` for add-ons that
    need to move to a compatible version for the target Kubernetes version.
  - **`deprecationDetails[]`** – the deprecated-API findings (next section).

### The `deprecationDetails` object — field by field

This is the heart of deprecated-API detection. Each entry has:

- **`usage`** – the **deprecated version of the resource** currently in use, e.g.
  `/apis/flowcontrol.apiserver.k8s.io/v1beta2/flowschemas`.
- **`replacedWith`** – the **newer resource version to migrate to**, e.g.
  `/apis/flowcontrol.apiserver.k8s.io/v1beta3/flowschemas`.
- **`startServingReplacementVersion`** – the Kubernetes version where the **replacement
  first became available** (so you know the earliest version you could migrate on), e.g.
  `1.26`.
- **`stopServingVersion`** – the Kubernetes version where the **deprecated version stops
  being served** (i.e. removed), e.g. `1.29`. This is the deadline.
- **`clientStats[]`** – **who is actually calling the deprecated API**, each entry being:
  - **`userAgent`** – the client's user agent (e.g. a controller, a `kubectl` version, a
    Helm operation, an in-house app's client-go string).
  - **`numberOfRequestsLast30Days`** – request count over the trailing 30 days.
  - **`lastRequestTime`** – timestamp of the most recent request seen.

The `clientStats` array is what makes Insights genuinely useful: it doesn't just say "a
deprecated API exists," it tells you **which client is hitting it and how often/recently**,
so you can trace the finding back to the workload, controller, or CI job responsible.

---

## 4. How the deprecated-API detection actually works

This is the key technical detail: **EKS detects deprecated API usage by observing real API
server traffic, not by inspecting stored object versions.**

- The scan is based on the **last 30 days of API requests** to the cluster's API server.
  A `PASSING` result literally reads *"No deprecated API usage detected within the last 30
  days."*
- Under the hood this is the same signal exposed by the Kubernetes metric
  **`apiserver_requested_deprecated_apis`** (available since Kubernetes v1.19) and the API
  server **audit-log annotation `k8s.io/deprecated: "true"`** (with
  `k8s.io/removed-release` naming the removal version). Insights packages that signal into
  curated findings with remediation.

### What this observation-based approach implies

- **A resource that exists but isn't being requested may not be flagged.** Detection keys
  off *requests* in the trailing 30 days. Something dormant, newly introduced, or only
  present in un-applied manifests won't show traffic.
- **False positives on live clusters are possible.** AWS's own guidance notes that
  live-cluster scanning tools can report false positives (for example, the control plane's
  own components touching an API). Scanning **static, rendered manifests** is more accurate
  for the "what will I deploy" question.
- **30-day window matters for cadence.** If a deprecated call happened 31+ days ago, or a
  workload that uses it hasn't run in the window, the check can read `PASSING` even though
  the manifest still references the old API. Don't treat a green Insight as proof your Git
  repos are clean.

This is why AWS positions Insights as the **first** check, not the only one — pair it with
a manifest/Helm scanner (`kubent`, `pluto`, `kubepug`) that reads your source.

### Verify the same signal yourself

Insights is essentially productizing these two raw sources, which you can query directly:

```bash
# 1. Prometheus-style counter of deprecated API requests (since k8s v1.19)
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
# apiserver_requested_deprecated_apis{group="policy",removed_release="1.25",
#   resource="podsecuritypolicies",version="v1beta1"} 1

# 2. Audit logs (requires control-plane audit logging enabled) — CloudWatch Logs Insights
CLUSTER="<cluster_name>"
QUERY_ID=$(aws logs start-query \
  --log-group-name /aws/eks/${CLUSTER}/cluster \
  --start-time $(date -v-30M "+%s") \
  --end-time $(date "+%s") \
  --query-string 'fields @message | filter `annotations.k8s.io/deprecated`="true"' \
  --query queryId --output text)
sleep 5
aws logs get-query-results --query-id "$QUERY_ID"
```

The audit-log records include the `objectRef` (resource/apiGroup/apiVersion), the `verb`,
the calling `user`/`userAgent`, and annotations `k8s.io/deprecated: "true"` and
`k8s.io/removed-release`, which map directly onto the `deprecationDetails` fields above.

---

## 5. Add-on compatibility checks

Beyond APIs, upgrade insights flag **EKS-managed add-ons** whose current version isn't
compatible with the target Kubernetes version, via `addonCompatibilityDetails[]`
(`name` + `compatibleVersions[]`). This typically covers the managed add-ons:

- VPC CNI (`vpc-cni`)
- CoreDNS (`coredns`)
- kube-proxy (`kube-proxy`)
- EBS / EFS CSI drivers
- and other EKS-managed add-ons

Same caveat as rollback: **only EKS-managed add-ons are evaluated.** Self-managed add-ons,
Helm-installed controllers, or a managed add-on whose version you pinned outside the
add-on lifecycle are invisible to this check. Validate those yourself.

---

## 6. Using it: CLI and API

Insights are available through the console (**Upgrade insights** tab of the cluster's
observability dashboard), the AWS SDKs, and two CLI/API operations.

### List insights

```bash
# All insights for a cluster
aws eks list-insights --region <region-code> --cluster-name <my-cluster>

# Filter to upgrade readiness for a specific target version
aws eks list-insights --region <region-code> --cluster-name <my-cluster> \
  --filter '{"categories":["UPGRADE_READINESS"],"kubernetesVersions":["1.35"]}'
```

The `--filter` (`InsightsFilter`) accepts `categories`, `kubernetesVersions`, and
`statuses` to narrow results. A typical `list-insights` item looks like:

```json
{
  "category": "UPGRADE_READINESS",
  "name": "Deprecated APIs removed in Kubernetes v1.29",
  "insightStatus": {
    "status": "PASSING",
    "reason": "No deprecated API usage detected within the last 30 days."
  },
  "kubernetesVersion": "1.29",
  "lastTransitionTime": 1698774710.0,
  "lastRefreshTime": 1700157422.0,
  "id": "123e4567-e89b-42d3-a456-579642341238",
  "description": "Checks for usage of deprecated APIs that are scheduled for removal in Kubernetes v1.29..."
}
```

### Describe a specific insight

```bash
aws eks describe-insight --region <region-code> \
  --cluster-name <my-cluster> --id <insight-id>
```

This returns the full object from Section 3, including `resources[]`,
`deprecationDetails[]` (with `clientStats`), `recommendation`, and
`addonCompatibilityDetails[]`.

Worked example of an `ERROR` finding's key fields:

```json
"resources": [
  { "insightStatus": { "status": "ERROR" },
    "kubernetesResourceUri": "/apis/policy/v1beta1/podsecuritypolicies/null" }
],
"deprecationDetails": [
  { "usage": "/apis/flowcontrol.apiserver.k8s.io/v1beta2/flowschemas",
    "replacedWith": "/apis/flowcontrol.apiserver.k8s.io/v1beta3/flowschemas",
    "stopServingVersion": "1.29",
    "startServingReplacementVersion": "1.26",
    "clientStats": [] }
],
"recommendation": "Update manifests and API clients to use newer Kubernetes APIs if applicable before upgrading."
```

### Refresh cadence

- Insights refresh **automatically every 24 hours**.
- You can **refresh on demand** (console or API) after fixing something to confirm the
  finding clears — useful because the underlying signal is the trailing-30-day request
  window, so you want to re-check once the offending client has stopped calling the old
  API.

---

## 7. Permissions model

You don't configure anything to enable Insights. EKS **automatically creates a cluster
access entry** in every EKS cluster that grants EKS permission to read the cluster and
generate insights, backed by the AWS-managed **`AmazonEKSClusterInsightsPolicy`**. There
is no setup, no agent, and no cost.

To *read* insights from the AWS side, an IAM principal needs the EKS permissions behind
`ListInsights` / `DescribeInsight` (e.g. `eks:ListInsights`, `eks:DescribeInsight`).

---

## 8. What Insights does NOT catch (the important gaps)

Insights only clears the **API-version and managed-add-on** dimensions of an upgrade, and
only based on **observed live traffic**. It is blind to a whole class of high-risk,
node- and runtime-level changes. For any real upgrade you must also handle:

- **Source manifests / Helm charts in Git.** Deprecated `apiVersion`s that exist only in
  un-applied manifests generate no API traffic, so Insights won't see them. Scan your
  repos with `kubent`, `pluto detect-files`, or `kubepug`/`kubectl deprecations`.
- **The 30-day / dormant-workload blind spot.** A workload that didn't run in the window,
  or a batch/CronJob that fires rarely, can leave a real problem unflagged.
- **Node and runtime changes.** cgroup v1 removal, kubelet flag removals (e.g.
  `--pod-infra-container-image`), removed/locked feature gates, containerd version support
  windows, RBAC behavior changes (e.g. `pods/exec` needing `create`) — none of these are
  API-server request patterns, so Insights doesn't cover them.
- **Self-managed add-ons and CSI sidecars.** Only EKS-managed add-ons are checked.
- **Client-side deprecations.** APIs *dropped by the kubectl client* but still served by
  the API server won't show as removed by a traffic-based scan.

### Recommended layered approach

1. **EKS Cluster Insights** (this doc) — the AWS-native, live-traffic baseline. Start here.
2. **Manifest/Helm scanners** — `kubent -t <version>`, `pluto detect-files` /
   `detect-all-in-cluster`, `kubepug` / `kubectl deprecations` against your **source**.
3. **Raw signals** — `apiserver_requested_deprecated_apis` metric and the
   `k8s.io/deprecated` audit-log annotation, for independent verification.
4. **Version-specific node/runtime checklist** — the actual changelog items that Insights
   can't see (cgroup, kubelet flags, feature gates, runtime versions).

---

## Summary

- **Cluster Insights** = automatic, recurring, no-cost, no-setup checks against an
  AWS-curated list, in three categories: **upgrade** (continuous, every cluster),
  **configuration** (hybrid nodes), and **rollback readiness** (7-day window, point-in-time).
- Its most valuable check is **deprecated/removed API usage**, detected from the **last 30
  days of real API server traffic**, with rich `deprecationDetails` telling you the
  deprecated version, its replacement, when the replacement became available, when the old
  version stops being served, and **which client is calling it** (`clientStats`).
- It also flags **EKS-managed add-on compatibility**.
- Statuses are `PASSING` / `WARNING` / `ERROR` / `UNKNOWN`; `ERROR`/`UNKNOWN` block
  rollbacks (overridable with `--force`), and should be treated as must-fix before upgrades.
- It is a **baseline, not a complete gate**: it doesn't see your Git manifests, dormant
  workloads, node/runtime changes, or self-managed add-ons. Layer manifest scanners and a
  version-specific checklist on top.

---

### Sources

- [Prepare for Kubernetes version upgrades and troubleshoot misconfigurations with cluster insights (Amazon EKS User Guide)](https://docs.aws.amazon.com/eks/latest/userguide/cluster-insights.html)
- [View cluster insights (Amazon EKS User Guide)](https://docs.aws.amazon.com/eks/latest/userguide/view-cluster-insights.html)
- [Best Practices for Cluster Upgrades — Identify and remediate removed API usage (Amazon EKS)](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-upgrades.html)
- [EKS API Reference: ListInsights](https://docs.aws.amazon.com/eks/latest/APIReference/API_ListInsights.html), [DescribeInsight](https://docs.aws.amazon.com/eks/latest/APIReference/API_DescribeInsight.html), [DeprecationDetail](https://docs.aws.amazon.com/eks/latest/APIReference/API_DeprecationDetail.html), [ClientStat](https://docs.aws.amazon.com/eks/latest/APIReference/API_ClientStat.html), [InsightStatus](https://docs.aws.amazon.com/eks/latest/APIReference/API_InsightStatus.html)
- [Accelerate the testing and verification of Amazon EKS upgrades with upgrade insights (AWS Containers Blog)](https://aws.amazon.com/blogs/containers/accelerate-the-testing-and-verification-of-amazon-eks-upgrades-with-upgrade-insights/)
