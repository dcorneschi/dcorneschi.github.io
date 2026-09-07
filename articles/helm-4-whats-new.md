# What's New in Helm 4

Helm 4 was released on **November 12, 2025** at KubeCon + CloudNativeCon North
America in Atlanta — the first new major version in six years, coinciding with
the project's 10th anniversary. This is a summary of the major changes, new
features, and breaking changes.

## Major Changes

### Server-Side Apply (SSA)

Helm 4 replaces the three-way merge method with Server-Side Apply, which has
been standard in Kubernetes since v1.22. Each field modification now has an
assigned owner, and conflicts generate explicit errors rather than silent
overwrites. The old three-way merge remains available via a flag for backward
compatibility.

### Enhanced Resource Monitoring

Integration with the kstatus library provides much more accurate health status
checking. Two new annotations — `helm.sh/readiness-success` and
`helm.sh/readiness-failure` — let you control deployment flow with fine
granularity, preventing race conditions where apps start before their
dependencies are ready.

### WebAssembly Plugin System

The plugin architecture was redesigned with an optional Wasm-based runtime for
better security. Three plugin types are now supported: CLI plugins, getter
plugins, and post-renderer plugins. Legacy plugins still work, but the new
system enables more customization of core functionality.

## New Features

- **OCI digest support** — install charts by digest for supply chain security:

  ```bash
  helm install myapp oci://registry.example.com/charts/app@sha256:abc123...
  ```

- **Multi-document values** — split complex values across multiple YAML files.
- **Custom template functions** — extend templating through plugins.
- **Content-based caching** — smarter cache using a chart content hash instead
  of name/version.
- **Stable SDK API** — breaking changes are complete, enabling the future
  Charts v3 format.

## Breaking Changes

- **Post-renderers** must be implemented as plugins — you can no longer pass an
  executable directly to `--post-renderer`.
- **Registry login** — `helm registry login` requires the domain name only, not
  a full URL.
- **CLI flags renamed** (old names deprecated but still work):
  - `--atomic` → `--rollback-on-failure`
  - `--force` → `--force-replace`

## Helm 3 vs Helm 4 Commands

The core command surface is unchanged — `install`, `upgrade`, `rollback`, etc.
work the same. The differences are in flags and a few behaviors. Old flags are
deprecated but still work in Helm 4.

| Task | Helm 3 | Helm 4 |
|------|--------|--------|
| Roll back automatically on a failed upgrade | `helm upgrade myapp ./chart --atomic` | `helm upgrade myapp ./chart --rollback-on-failure` |
| Force resource replacement | `helm upgrade myapp ./chart --force` | `helm upgrade myapp ./chart --force-replace` |
| Log in to an OCI registry | `helm registry login registry.example.com` (URL tolerated) | `helm registry login registry.example.com` (domain only, not a full URL) |
| Use a post-renderer | `helm install myapp ./chart --post-renderer ./script.sh` (executable) | `helm install myapp ./chart --post-renderer <plugin>` (must be a plugin) |
| Install a chart by OCI digest | not supported | `helm install myapp oci://registry.example.com/charts/app@sha256:abc123...` |
| Apply strategy (behavior, not a flag) | client-side three-way merge | Server-Side Apply (three-way merge available via a flag) |

## Performance Improvements

- Faster dependency resolution
- Content-based chart caching
- Clearer error messages
- Better OAuth and token support for private registries

## Compatibility

- Charts v2 remain fully compatible.
- Helm 3 is still supported and receiving updates (v3.20+).
- Backward compatibility was prioritized throughout the redesign.
- Requires Kubernetes 1.22+ for Server-Side Apply features.

## Sources

- [Helm 4 Released (official blog)](https://docs.helm.sh/blog/helm-4-released)
- [Helm project history](https://docs.helm.sh/history/)
- [CNCF: Helm Marks 10 Years With Release of Version 4](https://www.cncf.io/announcements/2025/11/12/helm-marks-10-years-with-release-of-version-4/)

_Content summarized from the sources above; rephrased for compliance with licensing restrictions._
