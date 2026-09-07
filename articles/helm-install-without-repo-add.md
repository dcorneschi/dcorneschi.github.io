# Installing Helm Charts Without `helm repo add`

You don't always need to run `helm repo add` before installing a chart. Helm can
pull a chart directly from an OCI registry, a packaged `.tgz` URL, or a repo URL
passed inline. The examples below use `metrics-server` as the running example.

## Method 1: OCI Registry (Recommended)

Modern Helm (v3.8+) treats OCI registries as first-class chart sources. Point at
the chart's OCI URL directly — no repo needed:

```bash
helm install metrics-server oci://registry-1.docker.io/bitnamicharts/metrics-server
```

## Method 2: Packaged Chart URL (.tgz)

Install straight from a published chart tarball:

```bash
helm install metrics-server https://kubernetes-sigs.github.io/metrics-server/charts/metrics-server-3.12.1.tgz
```

## Method 3: The --repo Flag

Pass the repository URL inline with `--repo` instead of registering it first:

```bash
helm install metrics-server metrics-server \
  --repo https://kubernetes-sigs.github.io/metrics-server/
```

## Method 4: Direct from a GitHub Release

Install from a chart `.tgz` attached to a GitHub release:

```bash
helm install metrics-server https://github.com/kubernetes-sigs/metrics-server/releases/download/metrics-server-helm-chart-3.12.1/metrics-server-3.12.1.tgz
```

## Customizing the Installation

Any of these methods accept the usual `--set` / `--values` flags. For example,
with the OCI method:

```bash
helm install metrics-server oci://registry-1.docker.io/bitnamicharts/metrics-server \
  --set args="{--kubelet-insecure-tls,--kubelet-preferred-address-types=InternalIP}"
```

## Which Method to Use

| Method | When to use |
|--------|-------------|
| OCI registry | Preferred for modern charts; no repo state, version pinned by tag |
| Packaged `.tgz` URL | Quick one-off install of a specific chart version |
| `--repo` flag | You know the repo URL but don't want to register it |
| GitHub release URL | Chart is published only as a release asset |

All four bypass the need to run `helm repo add` first. To pin a version with the
OCI or `--repo` methods, add `--version <x.y.z>`.

## Notes

- **OCI vs classic repos:** OCI URLs use the `oci://` scheme and are versioned by
  registry tag; classic repos serve an `index.yaml` that Helm reads to resolve
  versions.
- **Bitnami charts** (Method 1's example) are packaged independently from the
  upstream metrics-server chart, so values keys may differ from the
  `kubernetes-sigs` chart. Check the chart's own `values.yaml` before setting
  overrides.

## Related

- [Installing metrics-server on Kubernetes](articles/metrics-server-install.md)
- [Helm Cheatsheet](articles/helm-cheatsheet.md)

## Skills Practiced

- Installing charts from OCI registries, `.tgz` URLs, and inline repo URLs
- Skipping `helm repo add` for one-off or CI-friendly installs
- Passing `--set` overrides regardless of the chart source
- Knowing when OCI vs classic repo resolution applies
