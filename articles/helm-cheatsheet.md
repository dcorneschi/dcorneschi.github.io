<img src="/articles/images/helm-logo.svg" alt="Helm" width="150">

# Helm Cheatsheet

Comprehensive Helm reference guide featuring installation instructions, repository management, chart deployment, upgrade strategies, rollback procedures, and development best practices for Kubernetes package management.

## Installation

| Platform | Package Manager | Command |
|----------|----------------|---------|
| Linux | Script | `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \| bash` |
| macOS | Homebrew | `brew install helm` |
| Windows | Chocolatey | `choco install kubernetes-helm` |
| Windows | Scoop | `scoop install helm` |

### Verify Installation

```bash
helm version
```

### Shell Completion

| Shell | Command |
|-------|---------|
| Bash | `helm completion bash > /etc/bash_completion.d/helm` |
| Zsh | `helm completion zsh > ~/.zsh/completions/_helm` |

## Chart Structure

### Directory Layout

```
my-chart/
├── Chart.yaml              # Chart metadata (name, version, description)
├── values.yaml             # Default configuration values
├── values.schema.json      # JSON schema that validates values
├── templates/              # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl        # Template helpers (prefixed with _)
├── charts/                 # Dependency charts (optional)
└── README.md               # Documentation
```

### Chart Download Behavior

When you run `helm repo update`, Helm downloads only the index (metadata about available charts), not the actual chart packages. Charts are downloaded when you:

| Command | Behavior |
|---------|----------|
| `helm pull` | Download chart to local directory |
| `helm install` | Download and install (cached temporarily) |
| `helm upgrade` | Download new version (cached temporarily) |

## Repository Management

### Basic Commands

| Command | Details |
|---------|---------|
| `helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/` | Add a repository |
| `helm repo add myrepo https://charts.example.com --username user --password pass` | Add private repo with credentials |
| `helm repo add secure https://charts.secure.com --ca-file ./ca.crt --cert-file ./client.crt --key-file ./client.key` | Add repo with certificate auth |
| `helm repo update` | Update repository indexes |
| `helm repo update bitnami` | Update a specific repository |
| `helm repo update --debug` | Update with verbose/debug output |
| `helm repo update --fail-on-repo-update-fail` | Force update all repositories |
| `helm repo list` | List configured repositories |
| `helm repo list -o yaml` | List repositories in YAML format |
| `helm repo remove <repo-name>` | Remove a repository |
| `helm repo index <DIR>` | Generate an index file from charts found in a directory |
| `helm repo index <DIR> --merge` | Merge generated index with an existing index file |

### Search Commands

| Command | Details |
|---------|---------|
| `helm search repo metrics-server` | Search for charts in repositories |
| `helm search repo metrics-server/metrics-server --versions` | List all versions with default column width |
| `helm search repo metrics-server/metrics-server --versions --max-col-width 0` | List all versions with full width output (recommended) |
| `helm search repo metrics-server/metrics-server --output json` | Search with JSON output (useful for scripting) |
| `helm search repo metrics-server/metrics-server --output yaml` | Search with YAML output |
| `helm search hub metrics-server` | Search Helm Hub |

### Chart Information

| Command | Details |
|---------|---------|
| `helm show chart metrics-server/metrics-server` | Show chart metadata |
| `helm show chart metrics-server/metrics-server --version 3.12.2` | Show chart metadata for specific version |
| `helm show values metrics-server/metrics-server` | Show default values |
| `helm show values metrics-server/metrics-server --version 3.12.2` | Show default values for specific version |
| `helm show readme metrics-server/metrics-server` | Show README |
| `helm show readme metrics-server/metrics-server --version 3.12.2` | Show README for specific version |
| `helm show all metrics-server/metrics-server` | Show all info |

## Install Charts

### Basic Installation

| Command | Details |
|---------|---------|
| `helm install metrics-server metrics-server/metrics-server` | Basic installation (default namespace) |
| `helm install metrics-server metrics-server/metrics-server --set args={--kubelet-insecure-tls}` | Install without certificate validation |
| `helm install metrics-server metrics-server/metrics-server --namespace kube-system` | Install in specific namespace |
| `helm install metrics-server metrics-server/metrics-server -n kube-system --create-namespace` | Install and create the namespace |
| `helm install metrics-server metrics-server/metrics-server -n kube-system --create-namespace --set service.type=LoadBalancer` | Install with inline values |
| `helm install metrics-server metrics-server/metrics-server --set-string replicas=3` | Override value as string type |
| `helm install metrics-server metrics-server/metrics-server -f values.yaml` | Install with values file |
| `helm install metrics-server ./metrics-server` | Install from local chart (after `helm pull --untar`) |
| `helm install metrics-server ./metrics-server-3.12.2.tgz` | Install from packaged chart |
| `helm install metrics-server metrics-server/metrics-server --version 3.12.2` | Install with specific version |
| `helm install metrics-server/metrics-server --generate-name` | Auto-generate a release name |
| `helm install metrics-server metrics-server/metrics-server --verify` | Verify package signature before installing |
| `helm install metrics-server metrics-server/metrics-server --dependency-update` | Update missing dependencies before installing |

### Advanced Installation

| Command | Details |
|---------|---------|
| `helm install metrics-server metrics-server/metrics-server --wait --timeout 5m` | Wait for deployment to complete |
| `helm install metrics-server metrics-server/metrics-server --dry-run` | Dry run installation |
| `helm install metrics-server metrics-server/metrics-server --dry-run --debug` | Dry run & debug installation |
| `helm install metrics-server metrics-server/metrics-server --post-renderer ./kustomize-post-render.sh` | Install with post-render hook (e.g., kustomize) |
| `helm install metrics-server metrics-server/metrics-server --skip-crds` | Skip CRD installation |
| `helm install metrics-server metrics-server/metrics-server --disable-openapi-validation` | Disable OpenAPI validation |

## Upgrade Releases

### Basic Upgrade

| Command | Details |
|---------|---------|
| `helm upgrade metrics-server metrics-server/metrics-server -n kube-system` | Upgrade release |
| `helm upgrade metrics-server metrics-server/metrics-server -n kube-system --version 3.12.2` | Upgrade to specific version |
| `helm upgrade metrics-server metrics-server/metrics-server -n kube-system --set replicas=2` | Upgrade with inline values |
| `helm upgrade metrics-server metrics-server/metrics-server -n kube-system --values values.yaml` | Upgrade release with new values |

### Advanced Upgrade

| Command | Details |
|---------|---------|
| `helm upgrade metrics-server metrics-server/metrics-server -n kube-system --atomic` | Upgrade and rollback on failure |
| `helm upgrade metrics-server metrics-server/metrics-server -n kube-system --wait --timeout 5m` | Upgrade and wait for completion |
| `helm upgrade metrics-server metrics-server/metrics-server -n kube-system --dry-run` | Dry run upgrade |
| `helm upgrade metrics-server metrics-server/metrics-server -n kube-system --force` | Force update (recreate pods) |
| `helm upgrade --install metrics-server metrics-server/metrics-server -n kube-system` | Upgrade and install if not exists |
| `helm upgrade metrics-server metrics-server/metrics-server -n kube-system --dependency-update` | Update missing dependencies before upgrading |
| `helm upgrade metrics-server metrics-server/metrics-server -n kube-system --history-max=10` | Limit stored release history |
| `helm upgrade metrics-server metrics-server/metrics-server --kube-context my-cluster-context -n production` | Target a specific kube context |
| `helm upgrade --install metrics-server ./metrics-server -f <(helm get values metrics-server -n kube-system) --atomic` | Reinstall with last values |

## Rollback

| Command | Details |
|---------|---------|
| `helm history metrics-server -n kube-system` | Check upgrade history |
| `helm history metrics-server -n kube-system --max 10` | Show last 10 revisions |
| `helm rollback metrics-server -n kube-system` | Rollback to previous release |
| `helm rollback metrics-server -n kube-system 1` | Rollback to specific revision |
| `helm rollback metrics-server -n kube-system 1 --cleanup-on-fail` | Rollback with cleanup hooks |
| `helm rollback metrics-server -n kube-system 1 --wait --timeout 5m` | Rollback and wait for completion |

## Uninstall

| Command | Details |
|---------|---------|
| `helm uninstall metrics-server` | Uninstall release |
| `helm uninstall metrics-server -n kube-system` | Uninstall from specific namespace |
| `helm uninstall metrics-server -n kube-system --keep-history` | Keep release history (soft delete) |
| `helm uninstall metrics-server -n kube-system --no-hooks` | Skip hook execution during uninstall |
| `helm uninstall metrics-server --dry-run` | Dry run uninstall |
| `helm uninstall metrics-server --timeout 300s` | Uninstall with timeout |

## Download Charts

### Download to Current Directory

| Command | Details |
|---------|---------|
| `helm pull metrics-server/metrics-server` | Download chart package (tgz) to current directory |
| `helm pull metrics-server/metrics-server --verify` | Download and verify package signature |
| `helm pull metrics-server/metrics-server --untar` | Download and extract |
| `helm pull metrics-server/metrics-server --destination ~/helm-charts` | Download to specific location |
| `helm pull metrics-server/metrics-server --destination ~/helm-charts --untar` | Download and extract to specific location |
| `helm pull metrics-server/metrics-server --version 3.12.2 --destination ~/helm-charts` | Download specific version |

### Download Multiple Charts

```bash
mkdir -p ~/helm-charts/{metrics-server,nginx,postgres}
helm pull metrics-server/metrics-server --destination ~/helm-charts/metrics-server --untar
helm pull bitnami/nginx --destination ~/helm-charts/nginx --untar
helm pull bitnami/postgresql --destination ~/helm-charts/postgres --untar
```

## Create & Develop Charts

### Chart Development

| Command | Details |
|---------|---------|
| `helm create my-chart` | Create a new chart |
| `helm package my-chart` | Package chart into tarball |
| `helm package my-chart --version 1.2.0` | Package with a specific version |
| `helm package my-chart --version 1.2.0 --app-version 2.0.0` | Package with specific chart and app versions |
| `helm package my-chart -d dist` | Package to specific directory |
| `helm package my-chart --sign --key my-key --keyring ~/.gnupg/pubring.gpg` | Create a signed package |
| `helm lint my-chart` | Validate chart syntax |
| `helm lint my-chart --values values.yaml` | Validate chart values |
| `helm lint my-chart --strict` | Lint with strict mode (warnings are errors) |
| `helm lint my-chart --strict --with-subcharts` | Lint including subcharts |

### Template Rendering

| Command | Details |
|---------|---------|
| `helm template metrics-server metrics-server/metrics-server` | Template rendering (dry run) |
| `helm template metrics-server metrics-server/metrics-server --set replicas=2` | Template with inline values |
| `helm template metrics-server metrics-server/metrics-server --set-string key=value` | Template with string value override |
| `helm template metrics-server metrics-server/metrics-server --values values.yaml` | Template with values file |
| `helm template metrics-server metrics-server/metrics-server --debug` | Template with debug values |
| `helm template metrics-server metrics-server/metrics-server > metrics-server-rendered.yaml` | Template to file |

## Release Management

Release information is stored in Kubernetes, not on your local machine.

### List Releases

| Command | Details |
|---------|---------|
| `helm list -n kube-system` | List releases in specific namespace |
| `helm list --all` | Show all releases without any filter |
| `helm list -A` | List all releases across all namespaces (table format) |
| `helm list -A --output yaml` | List all releases in YAML format |
| `helm list -A --output json` | List all releases in JSON format |
| `helm list -A --output json \| jq '.[] \| .name'` | Get specific field |
| `helm list -l key1=value1,key2=value2` | Filter releases by label selector |
| `helm list --date` | Sort releases by date |
| `helm list --deployed` | Show only deployed releases |
| `helm list --failed` | Show only failed releases |
| `helm list --pending` | Show pending releases |
| `helm list --short` | Show only release names (useful for scripting) |
| `helm list --uninstalled` | Show uninstalled releases (requires `--keep-history` on uninstall) |
| `helm list --superseded` | Show superseded releases |

### Release Information

| Command | Details |
|---------|---------|
| `helm status metrics-server -n kube-system` | Show release details |
| `helm status metrics-server -n kube-system --revision 2` | Show release status at a specific revision |
| `helm get values metrics-server -n kube-system` | Get user-supplied values |
| `helm get values metrics-server -n kube-system --all` | Get all values used by release (including defaults) |
| `helm get values metrics-server -n kube-system --output json` | Get user-supplied values as JSON |
| `helm get values metrics-server -n kube-system --revision 2` | Get values from a specific revision |
| `helm get manifest metrics-server -n kube-system` | Get release manifest (rendered YAML) |
| `helm get manifest metrics-server -n kube-system --revision 2` | Get manifest at specific revision |
| `helm get notes metrics-server -n kube-system` | Get release notes |
| `helm get hooks metrics-server -n kube-system` | Download hooks for a release |
| `helm get all metrics-server -n kube-system` | Get all information about a named release |
| `helm history metrics-server -n kube-system` | Show release history |
| `helm history metrics-server -n kube-system --output json` | Show release history in JSON format |
| `helm history metrics-server -n kube-system --show-rollback-revision` | Show which revision each rollback targeted |

### Kubernetes Storage

| Command | Details |
|---------|---------|
| `kubectl get secrets -n <namespace> -l owner=helm` | Release data is stored in Kubernetes secrets |
| `helm diff revision metrics-server -n kube-system 1 2` | Compare releases (requires helm-diff plugin) |

### Backup Releases

```bash
# Backup a single release
helm get all metrics-server -n kube-system > backup-metrics-server.yaml

# Backup all releases across all namespaces
for release in $(helm list -A -q); do
  helm get all "$release" > "backup-$release.yaml"
done
```

## Environment Configuration

### View Helm Environment Variables

| Command | Details |
|---------|---------|
| `helm env` | Show all Helm environment variables |
| `helm env \| grep CACHE` | Check cache location |
| `helm env \| grep -E "(CACHE\|CONFIG\|DATA)"` | Common Helm paths |

### Set Custom Cache Location

```bash
# Set custom cache directory
export HELM_CACHE_HOME=~/my-custom-helm-cache

# Set custom config directory
export HELM_CONFIG_HOME=~/.config/helm

# Set custom data directory
export HELM_DATA_HOME=~/.local/share/helm

# Enable debug logging
export HELM_DEBUG=true

# Set default namespace for all Helm commands
export HELM_NAMESPACE=production

# Set OCI registry config path
export HELM_REGISTRY_CONFIG="${HOME}/.docker/config.json"

# Make permanent (add to ~/.bash_profile or ~/.zshrc)
echo 'export HELM_CACHE_HOME=~/my-custom-helm-cache' >> ~/.bash_profile
```

## Cache Management (macOS)

### Cache Locations

| Path | Description |
|------|-------------|
| `~/Library/Preferences/helm/` | Helm configuration directory |
| `~/Library/Caches/helm/repository/` | Repository cache |
| `~/Library/Preferences/helm/repositories.yaml` | Repository configuration |
| `~/Library/helm/plugins/` | Plugin directory |

### Cache Commands

| Command | Details |
|---------|---------|
| `ls -la ~/Library/Caches/helm/repository/` | List all files in repository cache |
| `ls -lh ~/Library/Caches/helm/repository/` | View with human-readable sizes |
| `ls -la ~/Library/Caches/helm/repository/*.yaml` | Show only YAML index files |
| `ls -la ~/Library/Caches/helm/repository/*.tgz` | Show only chart archives |
| `du -sh ~/Library/Caches/helm/` | Check cache directory size |
| `du -sh ~/Library/Caches/helm/repository/` | Check repository cache size |
| `du -h ~/Library/Caches/helm/repository/* \| sort -h` | List files by size |

### Cache Cleanup

| Command | Details |
|---------|---------|
| `rm -f ~/Library/Caches/helm/repository/*.tgz` | Remove all cached charts (keeps repository indexes) |
| `rm -f ~/Library/Caches/helm/repository/metrics-server-*.tgz` | Remove specific cached chart |
| `rm -rf ~/Library/Caches/helm/` | Clear entire cache (nuclear option) |
| `helm repo update` | After clearing, update repositories |

## Examples

### Extract and Modify Chart

```bash
# Download and extract chart
helm pull metrics-server/metrics-server --untar --destination ~/my-custom-charts

# Navigate to chart
cd ~/my-custom-charts/metrics-server

# Modify values or templates
vim values.yaml
vim templates/deployment.yaml

# Create custom chart package
cd ..
helm package metrics-server
# Creates: metrics-server-3.12.2.tgz

# Install custom chart
helm install metrics-server ./metrics-server-3.12.2.tgz
```

## Plugins

| Command | Details |
|---------|---------|
| `helm plugin install https://github.com/databus23/helm-diff` | Install helm-diff plugin |
| `helm plugin install https://github.com/jkroepke/helm-secrets` | Install helm-secrets plugin |
| `helm plugin install https://github.com/fabmation-gmbh/helm-whatup` | Install helm-whatup plugin (check outdated releases) |
| `helm plugin install https://github.com/chartmuseum/helm-push` | Install helm-push plugin (push charts to ChartMuseum) |
| `helm plugin install https://github.com/hypnoglow/helm-s3` | Install helm-s3 plugin (use AWS S3 as chart repo) |
| `helm plugin list` | List installed plugins |
| `helm plugin update diff` | Update a plugin |
| `helm plugin uninstall diff` | Uninstall a plugin |
| `helm diff upgrade metrics-server metrics-server/metrics-server --values values.yaml` | Preview what would change in an upgrade |
| `helm secrets enc secrets.yaml` | Encrypt secrets file |
| `helm secrets upgrade metrics-server ./charts/metrics-server -f values/prod.yaml -f secrets/values.enc.yaml --atomic` | Upgrade with encrypted secrets |
| `helm install metrics-server ./metrics-server -f secrets://secrets.yaml` | Install with encrypted secrets |

## Dependencies

| Command | Details |
|---------|---------|
| `helm dependency list my-chart` | List chart dependencies |
| `helm dependency update my-chart` | Download/update chart dependencies |
| `helm dependency build my-chart` | Rebuild the charts/ directory from Chart.lock |

### Declare Dependencies (Chart.yaml)

```yaml
apiVersion: v2
name: app
version: 0.1.0
dependencies:
  - name: redis
    version: 18.0.0
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

## Values Management

### helm show values vs helm get values

| Command | Details |
|---------|---------|
| `helm show values metrics-server/metrics-server` | Show default values from a **chart** (before installation) |
| `helm get values metrics-server` | Show **actual values used** for an installed release |
| `helm get values metrics-server --all` | Show all values including defaults for an installed release |

### Upgrade Values Flags

| Flag | Details |
|------|---------|
| `--reuse-values` | Reuse last release values, merge new overrides |
| `--reset-values` | Reset values to chart defaults, ignore previous values |
| `--reset-then-reuse-values` | Reset to chart defaults, then merge old values back (Helm 3.14.0+) |

```bash
# Reuse previous values, add a tweak
helm upgrade metrics-server metrics-server/metrics-server --reuse-values --set newEnv=production

# Override entirely — no old values retained
helm upgrade metrics-server metrics-server/metrics-server --reset-values --set newKey=newValue

# Reset to defaults then merge old values back
helm upgrade metrics-server metrics-server/metrics-server --reset-then-reuse-values --set version=2.0
```

**Important caveats:**
- `--reuse-values` disregards any changes in the new chart version's default values
- If you run `helm upgrade` **without** `--set`/`-f`, `--reuse-values` is used by default
- If you run `helm upgrade` **with** `--set`/`-f`, `--reset-values` is used by default

### Multiple Values Files

```bash
helm install metrics-server ./metrics-server \
  --values values-base.yaml \
  --values values-prod.yaml \
  --values values-secrets.yaml

# Inject sensitive values from files (avoid inline secrets)
helm upgrade metrics-server ./metrics-server --set-file db.password=./secrets/db_password.txt
```

## OCI Registries

| Command | Details |
|---------|---------|
| `helm registry login ghcr.io -u $USER --password-stdin` | Login to OCI registry |
| `helm registry logout ghcr.io` | Logout from OCI registry |
| `helm push metrics-server-3.12.2.tgz oci://ghcr.io/acme/charts` | Push chart to OCI registry |
| `helm install metrics-server oci://ghcr.io/acme/charts/metrics-server --version 3.12.2` | Install from OCI registry |

## Chart Signing & Security

| Command | Details |
|---------|---------|
| `helm package my-chart --sign --key my-key --keyring ~/.gnupg/pubring.gpg` | Create a signed package |
| `helm verify metrics-server-3.12.2.tgz` | Verify chart signature |
| `helm install metrics-server ./metrics-server-3.12.2.tgz --verify` | Install from signed chart |

## CI Directory

The `ci/` directory in a Helm chart contains values files used for CI/CD testing:

```bash
# chart-testing tool automatically tests all ci/*.yaml files
ct lint --charts ./my-chart
ct install --charts ./my-chart

# Or manually test with specific values files
helm lint ./my-chart -f ./my-chart/ci/default-values.yaml
helm install test-release ./my-chart -f ./my-chart/ci/minimal-values.yaml --dry-run
```

## Hooks

### Hook Annotations

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ include "app.fullname" . }}-migrate"
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["./migrate.sh"]
```

### Available Hook Types

`pre-install`, `post-install`, `pre-delete`, `post-delete`, `pre-upgrade`, `post-upgrade`, `pre-rollback`, `post-rollback`, `test`

### Chart Tests

```yaml
# templates/tests/test-conn.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "app.fullname" . }}-test"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: curl
      image: curlimages/curl
      args: ["-fsS", "http://{{ include \"app.fullname\" . }}"]
  restartPolicy: Never
```

```bash
helm test metrics-server --logs
```

## Templating

### Common Template Patterns

```yaml
# Conditionals
{{- if .Values.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
...
{{- end }}

# Loops and defaults
{{- range $k, $v := .Values.extraEnv }}
- name: {{ $k }}
  value: {{ $v | quote }}
{{- end }}

# Default values
{{- $replicas := default 2 .Values.replicaCount -}}

# Helper templates (_helpers.tpl)
{{- define "app.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```

### Selective Render (debug a single file)

| Command | Details |
|---------|---------|
| `helm template metrics-server ./metrics-server --show-only templates/deployment.yaml` | Render a single template file |
| `helm template metrics-server ./metrics-server --kube-version=1.28.0` | Render against a specific Kubernetes version |
| `helm template metrics-server ./metrics-server \| kubectl diff -f -` | Preview rendered manifests diff |

## Check for Chart Updates

```bash
# 1. Update repos
helm repo update

# 2. Check installed
helm list -A

# 3. Check available versions
helm search repo metrics-server/metrics-server --versions

# 4. Compare values between versions
helm show values metrics-server/metrics-server --version 3.12.0 > old-values.yaml
helm show values metrics-server/metrics-server --version 3.12.2 > new-values.yaml
diff old-values.yaml new-values.yaml
```

### Automated Update Check Script

```bash
#!/bin/bash
helm repo update > /dev/null 2>&1

helm list -A --output json | jq -r '.[] | "\(.name) \(.namespace) \(.chart)"' | while read name namespace chart; do
    chart_name=$(echo $chart | sed 's/-[0-9].*//')
    installed_version=$(echo $chart | sed 's/.*-\([0-9].*\)/\1/')

    # Search for the chart and prefer the repo where repo_name matches chart_name
    # (e.g., metrics-server/metrics-server over bitnami/metrics-server)
    latest=$(helm search repo "$chart_name" --output json 2>/dev/null | \
      jq -r --arg cn "$chart_name" '
        [.[] | select(.name | endswith("/"+$cn))] |
        (map(select(.name | startswith($cn+"/"))) | first) //
        first // empty
      ')
    latest_version=$(echo "$latest" | jq -r '.version // "unknown"')
    repo_chart=$(echo "$latest" | jq -r '.name // "unknown"')

    echo "Release: $name (Namespace: $namespace)"
    echo "  Installed: $chart"
    echo "  Latest: $repo_chart $latest_version"

    if [ "$installed_version" != "$latest_version" ]; then
        echo "  ⚠️  UPDATE AVAILABLE"
    else
        echo "  ✓ Up to date"
    fi
    echo ""
done
```

### Interactive Upgrade Script

```bash
#!/bin/bash
helm repo update > /dev/null 2>&1

releases=()
names=()
namespaces=()
chart_names=()
installed_versions=()
latest_versions=()
repo_charts=()

# Gather release info
while IFS= read -r line; do
    name=$(echo "$line" | jq -r '.name')
    namespace=$(echo "$line" | jq -r '.namespace')
    chart=$(echo "$line" | jq -r '.chart')
    chart_name=$(echo "$chart" | sed 's/-[0-9].*//')
    installed_version=$(echo "$chart" | sed 's/.*-\([0-9].*\)/\1/')

    latest=$(helm search repo "$chart_name" --output json 2>/dev/null | \
      jq -r --arg cn "$chart_name" '
        [.[] | select(.name | endswith("/"+$cn))] |
        (map(select(.name | startswith($cn+"/"))) | first) //
        first // empty
      ')
    latest_version=$(echo "$latest" | jq -r '.version // "unknown"')
    repo_chart=$(echo "$latest" | jq -r '.name // "unknown"')

    releases+=("$name")
    names+=("$name")
    namespaces+=("$namespace")
    chart_names+=("$chart_name")
    installed_versions+=("$installed_version")
    latest_versions+=("$latest_version")
    repo_charts+=("$repo_chart")
done < <(helm list -A --output json | jq -c '.[]')

# Display all releases
echo "=========================================="
echo "  Installed Helm Releases"
echo "=========================================="
echo ""

for i in "${!releases[@]}"; do
    status="✓"
    if [ "${installed_versions[$i]}" != "${latest_versions[$i]}" ]; then
        status="⚠️  UPDATE"
    fi
    printf "  %d) %-20s %-15s %s → %s  %s\n" \
        $((i+1)) "${names[$i]}" "${namespaces[$i]}" \
        "${installed_versions[$i]}" "${latest_versions[$i]}" "$status"
done

echo ""
echo "=========================================="
echo ""
read -p "Select release to upgrade (1-${#releases[@]}) or 'q' to quit: " choice

if [[ "$choice" == "q" || -z "$choice" ]]; then
    echo "Exiting."
    exit 0
fi

if [[ "$choice" -lt 1 || "$choice" -gt ${#releases[@]} ]]; then
    echo "Invalid selection."
    exit 1
fi

idx=$((choice - 1))
name="${names[$idx]}"
namespace="${namespaces[$idx]}"
repo_chart="${repo_charts[$idx]}"
latest_version="${latest_versions[$idx]}"

echo ""
echo "Upgrading $name in namespace $namespace to $repo_chart $latest_version..."
echo ""
read -p "Proceed? (y/n): " confirm

if [[ "$confirm" != "y" ]]; then
    echo "Cancelled."
    exit 0
fi

helm upgrade "$name" "$repo_chart" \
    --namespace "$namespace" \
    --version "$latest_version" \
    --reuse-values

echo ""
echo "Done. Release status:"
helm status "$name" -n "$namespace" | head -10
```

## Real-World Workflows

### Blue-Green Deployment

```bash
# Deploy blue (v1)
helm install my-app-blue ./my-app \
  --set image.tag=v1 \
  --namespace production

# Deploy green (v2)
helm install my-app-green ./my-app \
  --set image.tag=v2 \
  --namespace production

# Switch traffic (via service selector or ingress)
kubectl patch service my-app -p '{"spec":{"selector":{"version":"green"}}}'

# Cleanup blue when stable
helm uninstall my-app-blue --namespace production
```

### Canary Deployment

```bash
# Deploy stable version
helm install my-app ./my-app \
  --set replicaCount=3 \
  --set image.tag=stable

# Deploy canary with 1 replica
helm install my-app-canary ./my-app \
  --set replicaCount=1 \
  --set image.tag=canary

# Monitor canary metrics
kubectl top pods -l app=my-app

# Promote canary to stable
helm upgrade my-app ./my-app \
  --set image.tag=canary \
  --reuse-values

# Remove canary
helm uninstall my-app-canary
```

### Upgrade with Downtime Prevention

```bash
helm upgrade metrics-server metrics-server/metrics-server -n kube-system \
  --set podDisruptionBudget.minAvailable=1 \
  --set strategy.type=RollingUpdate \
  --set strategy.rollingUpdate.maxUnavailable=0 \
  --wait --timeout 10m
```

### Production Deployment

```bash
helm upgrade --install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --create-namespace \
  --values values-prod.yaml \
  --wait \
  --timeout 10m \
  --atomic
```

### Multi-Environment Management

```bash
# Development
helm upgrade --install metrics-server metrics-server/metrics-server -n dev --values values-dev.yaml

# Staging
helm upgrade --install metrics-server metrics-server/metrics-server -n staging --values values-staging.yaml

# Production
helm upgrade --install metrics-server metrics-server/metrics-server -n production --values values-prod.yaml
```

## Helm Install Lifecycle

What happens when you run `helm install`:

1. **Load & validate** — Reads `Chart.yaml`, `values.yaml`, `templates/`, merges values
2. **Render templates** — Go template engine substitutes `.Values`, `.Release`, `.Chart`
3. **API server** — Uses kubeconfig context to authenticate (Helm 3 is client-only)
4. **Apply manifests** — Sends rendered manifests to API server, passes admission controllers
5. **Store release** — Stores release state as a Secret (`helm.sh/release.v1`) in the namespace
6. **Controllers reconcile** — Deployment/DaemonSet controllers create pods, scheduler assigns nodes
7. **Service provisioning** — If `type: LoadBalancer`, cloud controller creates load balancer
8. **Endpoints converge** — EndpointSlice controller populates endpoints with ready pod IPs

## EKS-Specific Repositories

| Command | Details |
|---------|---------|
| `helm repo add eks https://aws.github.io/eks-charts` | AWS Load Balancer Controller |
| `helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver` | AWS EBS CSI Driver |
| `helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver` | AWS EFS CSI Driver |
| `helm repo add autoscaler https://kubernetes.github.io/autoscaler` | Cluster Autoscaler |

## Troubleshooting

| Command | Details |
|---------|---------|
| `helm install metrics-server metrics-server/metrics-server --debug --dry-run` | Verbose dry run output |
| `helm install metrics-server metrics-server/metrics-server --debug --dry-run -v=3` | Verbose output with debug level |
| `helm template metrics-server ./metrics-server --debug --validate` | Template debugging with validation |
| `helm status metrics-server --show-resources` | Show release resources |
| `helm lint ./my-chart --strict --with-subcharts` | Strict lint with subcharts |
| `helm list --failed` | List failed releases |
| `helm list --failed -q \| xargs -I {} helm delete {}` | Clean up all failed releases |
| `helm template metrics-server ./metrics-server \| kubeconform -strict -kubernetes-version 1.28 -` | Validate rendered resources |

### Common Issues

```bash
# Release stuck in pending-install
helm rollback metrics-server 0

# Resource conflicts
kubectl get all -l app.kubernetes.io/managed-by=Helm

# Force delete stuck release
helm delete metrics-server --no-hooks
kubectl delete secret -l owner=helm,name=metrics-server -n kube-system
```

## Useful Aliases

```bash
# Add to ~/.bashrc or ~/.zshrc
alias h='helm'
alias hls='helm list'
alias hlsa='helm list --all-namespaces'
alias hs='helm status'
alias hi='helm install'
alias hu='helm upgrade'
alias hr='helm rollback'
alias hd='helm delete'
alias ht='helm template'
alias hg='helm get'
```

### Useful Environment Variables

```bash
export HELM_NAMESPACE=production    # Set default namespace
export HELM_CACHE_HOME=~/my-cache  # Set cache directory
export KUBECONFIG=~/.kube/config   # Set kubeconfig
```

## Handy Flags Reference

| Flag | Details |
|------|---------|
| `--atomic` | Roll back automatically on failure (renamed `--rollback-on-failure` in Helm 4) |
| `--force` | Force resource update via delete/recreate (renamed `--force-replace` in Helm 4) |
| `--wait` / `--timeout` | Block until resources are ready |
| `--history-max` | Cap stored revisions |
| `--set` / `--set-file` / `--set-string` | Control value types precisely |
| `--show-only` | Render a single template file for debugging |
| `--kube-version` | Render as if targeting a specific cluster version |
| `--kube-context` | Target a specific kube context |
| `--skip-crds` | Skip CRD installation |
| `--no-hooks` | Skip hook execution |
| `--reuse-values` | Reuse values from previous release |
| `--reset-values` | Reset values to chart defaults |
| `--reset-then-reuse-values` | Reset to defaults, then merge old values back (Helm 3.14+) |
| `--post-renderer` | Pipe rendered manifests through an external command |
| `--disable-openapi-validation` | Disable OpenAPI schema validation |
| `--dependency-update` | Update dependencies before install/upgrade |
| `--generate-name` | Auto-generate a release name |

## Resources

* [https://helm.sh](https://helm.sh)
* [https://github.com/helm/helm](https://github.com/helm/helm)
* [https://artifacthub.io](https://artifacthub.io)
* [https://helm.sh/docs](https://helm.sh/docs)
