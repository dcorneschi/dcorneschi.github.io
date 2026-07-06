<p align="center">
  <img src="/cheatsheets/helm/images/helm-logo.svg" alt="helm logo" width="201"/>
</p>

<h1 align="center">Helm Cheatsheet</h1>

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
| `helm repo update` | Update repository indexes |
| `helm repo update --fail-on-repo-update-fail` | Force update all repositories |
| `helm repo list` | List configured repositories |
| `helm repo list -o yaml` | List repositories in YAML format |
| `helm repo remove <repo-name>` | Remove a repository |

### Search Commands

| Command | Details |
|---------|---------|
| `helm search repo traefik` | Search for charts in repositories |
| `helm search repo traefik/traefik --versions` | List all versions with default column width |
| `helm search repo traefik/traefik --versions --max-col-width 0` | List all versions with full width output (recommended) |
| `helm search hub metrics-server` | Search Helm Hub |

### Chart Information

| Command | Details |
|---------|---------|
| `helm show chart traefik/traefik` | Show chart metadata |
| `helm show values traefik/traefik` | Show default values |
| `helm show readme traefik/traefik` | Show README |
| `helm show all traefik/traefik` | Show all info |

## Install Charts

### Basic Installation

| Command | Details |
|---------|---------|
| `helm install metrics-server metrics-server/metrics-server` | Basic installation (default namespace) |
| `helm install metrics-server metrics-server/metrics-server --namespace kube-system` | Install in specific namespace |
| `helm install traefik traefik/traefik -n traefik --create-namespace` | Install and create the namespace |
| `helm install traefik traefik/traefik -n traefik --create-namespace --set service.type=LoadBalancer` | Install with inline values |
| `helm install traefik traefik/traefik -f values.yaml` | Install with values file |
| `helm install traefik ./traefik` | Install from local chart |
| `helm install traefik ./traefik-37.3.0.tgz` | Install from packaged chart |
| `helm install metrics-server metrics-server/metrics-server --version 3.12.2` | Install with specific version |

### Advanced Installation

| Command | Details |
|---------|---------|
| `helm install metrics-server metrics-server/metrics-server --wait --timeout 5m` | Wait for deployment to complete |
| `helm install traefik traefik/traefik --dry-run` | Dry run installation |
| `helm install traefik traefik/traefik --dry-run --debug` | Dry run & debug installation |

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

## Rollback

| Command | Details |
|---------|---------|
| `helm history metrics-server -n kube-system` | Check upgrade history |
| `helm rollback metrics-server -n kube-system` | Rollback to previous release |
| `helm rollback metrics-server -n kube-system 1` | Rollback to specific revision |
| `helm rollback metrics-server -n kube-system 1 --cleanup-on-fail` | Rollback with cleanup hooks |

## Uninstall

| Command | Details |
|---------|---------|
| `helm uninstall metrics-server` | Uninstall release |
| `helm uninstall metrics-server -n kube-system` | Uninstall from specific namespace |
| `helm uninstall metrics-server -n kube-system --keep-history` | Keep release history (soft delete) |

## Download Charts

### Download to Current Directory

| Command | Details |
|---------|---------|
| `helm pull traefik/traefik` | Download chart package (tgz) to current directory |
| `helm pull traefik/traefik --untar` | Download and extract |
| `helm pull traefik/traefik --destination ~/helm-charts` | Download to specific location |
| `helm pull traefik/traefik --destination ~/helm-charts --untar` | Download and extract to specific location |
| `helm pull traefik/traefik --version 27.0.0 --destination ~/helm-charts` | Download specific version |

### Download Multiple Charts

```bash
mkdir -p ~/helm-charts/{traefik,nginx,postgres}
helm pull traefik/traefik --destination ~/helm-charts/traefik --untar
helm pull bitnami/nginx --destination ~/helm-charts/nginx --untar
helm pull bitnami/postgresql --destination ~/helm-charts/postgres --untar
```

## Create & Develop Charts

### Chart Development

| Command | Details |
|---------|---------|
| `helm create my-chart` | Create a new chart |
| `helm package my-chart` | Package chart into tarball |
| `helm lint my-chart` | Validate chart syntax |
| `helm lint my-chart --values values.yaml` | Validate chart values |

### Template Rendering

| Command | Details |
|---------|---------|
| `helm template metrics-server metrics-server/metrics-server` | Template rendering (dry run) |
| `helm template metrics-server metrics-server/metrics-server --set replicas=2` | Template with inline values |
| `helm template metrics-server metrics-server/metrics-server --values values.yaml` | Template with values file |
| `helm template metrics-server metrics-server/metrics-server --debug` | Template with debug values |
| `helm template metrics-server metrics-server/metrics-server > metrics-server-rendered.yaml` | Template to file |

## Release Management

Release information is stored in Kubernetes, not on your local machine.

### List Releases

| Command | Details |
|---------|---------|
| `helm list -n argo` | List releases in specific namespace |
| `helm list -A` | List all releases across all namespaces (table format) |
| `helm list -A --output yaml` | List all releases in YAML format |
| `helm list -A --output json` | List all releases in JSON format |
| `helm list -A --output json \| jq '.[] \| .name'` | Get specific field |

### Release Information

| Command | Details |
|---------|---------|
| `helm status argo -n argo` | Show release details |
| `helm get values argo -n argo` | Get user-supplied values |
| `helm get values argo -n argo --all` | Get all values used by release (including defaults) |
| `helm get manifest argo -n argo` | Get release manifest (rendered YAML) |
| `helm get manifest metrics-server -n kube-system --revision 2` | Get manifest at specific revision |
| `helm get notes argo -n argo` | Get release notes |
| `helm get all argo -n argo` | Get all information about a named release |
| `helm history argo -n argo` | Show release history |
| `helm history argo -n argo --output json` | Show release history in JSON format |

### Kubernetes Storage

| Command | Details |
|---------|---------|
| `kubectl get secrets -n <namespace> -l owner=helm` | Release data is stored in Kubernetes secrets |
| `helm diff revision metrics-server -n kube-system 1 2` | Compare releases (requires helm-diff plugin) |

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
| `rm -f ~/Library/Caches/helm/repository/traefik-*.tgz` | Remove specific cached chart |
| `rm -rf ~/Library/Caches/helm/` | Clear entire cache (nuclear option) |
| `helm repo update` | After clearing, update repositories |

## Examples

### Extract and Modify Chart

```bash
# Download and extract chart
helm pull traefik/traefik --untar --destination ~/my-custom-charts

# Navigate to chart
cd ~/my-custom-charts/traefik

# Modify values or templates
vim values.yaml
vim templates/deployment.yaml

# Create custom chart package
cd ..
helm package traefik
# Creates: traefik-27.0.0.tgz

# Install custom chart
helm install my-traefik ./traefik-27.0.0.tgz
```

## Resources

* [https://helm.sh](https://helm.sh)
* [https://github.com/helm/helm](https://github.com/helm/helm)
* [https://artifacthub.io](https://artifacthub.io)
* [https://helm.sh/docs](https://helm.sh/docs)
