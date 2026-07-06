<p align="center">
  <img src="/cheatsheets/k9s/images/k9s-logo.png" alt="k9s logo" width="200"/>
</p>

<h1 align="center">k9s Cheatsheet</h1>

Complete k9s reference guide featuring launch options, resource navigation, log viewing, filtering, sorting, and cluster management commands for efficient Kubernetes terminal UI workflows.

## Installation

| Platform | Package Manager | Command |
|----------|----------------|---------|
| macOS / Linux | Homebrew | `brew install derailed/k9s/k9s` |
| Linux (Snap) | Snap | `sudo snap install k9s --devmode` |
| Linux (PacMan) | PacMan | `sudo pacman -S k9s` |
| Go | go install | `go install github.com/derailed/k9s@latest` |
| Any | Binary | Download from [GitHub Releases](https://github.com/derailed/k9s/releases) |

### Verify Installation

```bash
k9s version
```

## Launch & General

| Command | Details |
|---------|---------|
| `k9s` | Start k9s (default context) |
| `k9s -n <namespace>` | Start in a specific namespace |
| `k9s --context <ctx>` | Start with a specific kubeconfig context |
| `k9s --readonly` | Start in read-only mode |
| `k9s --kubeconfig <path>` | Use a specific kubeconfig file |
| `k9s --headless` | Start without header |
| `k9s --logoless` | Start without logo |
| `k9s --crumbsless` | Start without crumbs |
| `k9s -c <resource>` | Start directly in a resource view (e.g. `k9s -c pod`) |
| `:q` / `Ctrl+c` | Quit k9s |
| `?` | Show help / keybindings |
| `Ctrl+a` | Show all available resource aliases |

## Navigation

### Resource Commands

k9s accepts singular, plural, short-name, or alias for any resource:

| Command | Details |
|---------|---------|
| `:pod` | Jump to Pods view |
| `:pod ns-x` | View pods in namespace `ns-x` |
| `:pod /fred` | View pods filtered by `fred` |
| `:pod app=fred,env=dev` | View pods matching labels `app=fred` and `env=dev` |
| `:pod @ctx1` | View pods in context `ctx1` (switches context) |

### Core Resources

| Command | Details |
|---------|---------|
| `:pod` | Jump to Pods view |
| `:svc` | Jump to Services view |
| `:deploy` | Jump to Deployments view |
| `:ns` | Jump to Namespaces view |
| `:node` | Jump to Nodes view |
| `:sec` | Jump to Secrets view |
| `:cm` | Jump to ConfigMaps view |
| `:ing` | Jump to Ingresses view |
| `:ep` | Jump to Endpoints view |
| `:rs` | Jump to ReplicaSets view |

### Workloads

| Command | Details |
|---------|---------|
| `:cj` | Jump to CronJobs view |
| `:job` | Jump to Jobs view |
| `:ds` | Jump to DaemonSets view |
| `:sts` | Jump to StatefulSets view |
| `:rc` | Jump to ReplicationControllers view |

### Storage

| Command | Details |
|---------|---------|
| `:pv` | Jump to PersistentVolumes view |
| `:pvc` | Jump to PersistentVolumeClaims view |
| `:sc` | Jump to StorageClasses view |

### RBAC & Security

| Command | Details |
|---------|---------|
| `:sa` | Jump to ServiceAccounts view |
| `:rb` | Jump to RoleBindings view |
| `:crb` | Jump to ClusterRoleBindings view |
| `:role` | Jump to Roles view |
| `:cr` | Jump to ClusterRoles view |
| `:np` | Jump to NetworkPolicies view |

### Scaling & Config

| Command | Details |
|---------|---------|
| `:hpa` | Jump to HorizontalPodAutoscalers view |
| `:events` | Jump to Events view |
| `:limits` | Jump to LimitRanges view |
| `:quota` | Jump to ResourceQuotas view |

### Custom Resources

| Command | Details |
|---------|---------|
| `:crd` | Jump to CustomResourceDefinitions view |
| `:<crd-short-name>` | Jump to any CRD by its short name or full name |

## Resource Actions

| Key | Details |
|-----|---------|
| `Enter` | Describe / drill into resource |
| `d` | Describe selected resource |
| `y` | View YAML manifest |
| `e` | Edit resource (opens in editor) |
| `Ctrl+d` | Delete resource (TAB and ENTER to confirm) |
| `l` | View logs for pod/container |
| `p` | View logs for previous container instance |
| `s` | Shell into a container |
| `a` | Attach to a container |
| `f` | Port-forward |
| `Ctrl+k` | Kill a pod (no confirmation, equivalent to `kubectl delete --now`) |
| `b` | Run benchmark (if supported) |
| `n` | Set namespace |
| `r` | Refresh view |
| `Ctrl+s` | Save resource YAML to file |

## Logs View

| Key | Details |
|-----|---------|
| `l` | Toggle logs view |
| `0` | Tail all logs (since start) |
| `1` | Tail logs (1 minute) |
| `2` | Tail logs (5 minutes) |
| `3` | Tail logs (15 minutes) |
| `4` | Tail logs (30 minutes) |
| `5` | Tail logs (1 hour) |
| `t` | Toggle timestamps |
| `w` | Toggle line wrap |
| `s` | Toggle auto-scroll |
| `c` | Copy selected log line |
| `f` | Toggle fullscreen |
| `Esc` | Back to previous view |

## Filtering & Searching

| Key | Details |
|-----|---------|
| `/` | Filter / search resources by name (regex supported, e.g. `fred\|blee`) |
| `/!<term>` | Inverse filter (exclude matches) |
| `/-l <label>=<value>` | Filter by label selector |
| `/-f <term>` | Fuzzy find a resource |
| `/-f <field>=<value>` | Filter by field selector |
| `Esc` | Clear filter |

## Sorting & Display

| Key | Details |
|-----|---------|
| `Shift+<column key>` | Sort by column (shown in header) |
| `Ctrl+e` | Toggle header columns |
| `Ctrl+w` | Toggle wide view |

## Namespace

| Key | Details |
|-----|---------|
| `:ns` | Switch to namespace view |
| `0` (in any view) | Show all namespaces |

## Benchmarking & Pulses

| Key | Details |
|-----|---------|
| `:pulse` / `:pu` | Show cluster pulse (overview dashboard) |
| `:xray <resource> [namespace]` | X-ray a resource tree (e.g. `:xray deploy`, `:xray svc ns-x`). Supports: po, svc, dp, rs, sts, ds |
| `:popeye` | Run Popeye cluster sanitizer (if installed) |

## Context & Cluster

| Key | Details |
|-----|---------|
| `:ctx` | Switch Kubernetes context |
| `:ctx <context-name>` | Switch directly to a specific context (returns to last used view) |
| `:screendump` / `:sd` | View all saved resources |
| `:aliases` | Show all resource aliases |
| `:dir <path>` | Browse screendumps directory |

## Themes / Skins

k9s supports custom color themes via skin files. A collection of community skins is available at [k9s/skins](https://github.com/derailed/k9s/tree/master/skins).

### Finding Your Paths

The actual config/skins paths vary by OS. Run `k9s info` to see yours:

```bash
k9s info
```

| Platform | Config | Skins |
|----------|--------|-------|
| macOS | `~/Library/Application Support/k9s/config.yaml` | `~/Library/Application Support/k9s/skins/` |
| Linux (XDG) | `~/.local/share/k9s/config.yaml` | `~/.local/share/k9s/skins/` |
| Linux (legacy) | `~/.config/k9s/config.yaml` | `~/.config/k9s/skins/` |

### Setup

| Step | Details |
|------|---------|
| Find skin directory | Run `k9s info` and check the `Skins` path |
| Download a skin | Copy a `.yaml` skin file into the skins directory |
| Activate skin | Set `skin` in your `config.yaml` (see below) |
| Apply changes | Restart k9s |

### Activate a Skin

Edit your k9s `config.yaml` (path from `k9s info`):

> **Important:** On macOS, k9s v0.50+ uses `~/Library/Application Support/k9s/config.yaml` — not `~/.config/k9s/config.yaml`. Always run `k9s info` to confirm the correct path.

```yaml
k9s:
  ui:
    skin: dracula
```

### Download All Community Skins

Use `k9s info` to find your skins directory, then:

```bash
# macOS (k9s v0.50+)
SKINS_DIR="$HOME/Library/Application Support/k9s/skins"
mkdir -p "$SKINS_DIR" && curl -sL "https://api.github.com/repos/derailed/k9s/contents/skins" | grep '"download_url"' | cut -d '"' -f 4 | while read url; do curl -sL -o "$SKINS_DIR/$(basename "$url")" "$url"; done

# Linux
SKINS_DIR="$HOME/.local/share/k9s/skins"
mkdir -p "$SKINS_DIR" && curl -sL "https://api.github.com/repos/derailed/k9s/contents/skins" | grep '"download_url"' | cut -d '"' -f 4 | while read url; do curl -sL -o "$SKINS_DIR/$(basename "$url")" "$url"; done

# Not sure? Run `k9s info` and check the "Skins" line for your actual path.
```

### Popular Skins

| Skin | Description |
|------|-------------|
| `dracula` | Dark theme based on Dracula color palette |
| `monokai` | Monokai-inspired dark theme |
| `gruvbox-dark` | Retro groove dark color scheme |
| `gruvbox-light` | Retro groove light color scheme |
| `nord` | Arctic, north-bluish color palette |
| `one-dark` | Atom One Dark inspired theme |
| `solarized-dark` | Solarized dark color scheme |
| `solarized-light` | Solarized light color scheme |
| `catppuccin-mocha` | Soothing pastel dark theme |
| `rose-pine` | Soho vibes dark theme |
| `tokyo-night` | Clean dark theme inspired by Tokyo night |
| `transparent` | Transparent background (uses terminal colors) |

## Configuration

Run `k9s info` to find the exact paths for your system. Common locations:

| Platform | Config Directory |
|----------|-----------------|
| macOS (v0.50+) | `~/Library/Application Support/k9s/` |
| Linux | `~/.local/share/k9s/` |

> **Tip:** Always run `k9s info` to see the exact paths for your version and OS. Older versions may use `~/.config/k9s/`.

| File | Details |
|------|---------|
| `config.yaml` | Main k9s configuration |
| `skins/` | Custom skin/theme files |
| `hotkeys.yaml` | Custom hotkey definitions |
| `plugins.yaml` | Plugin definitions |
| `aliases.yaml` | Custom resource aliases |
| `views.yaml` | Custom column definitions per resource |

### Custom Hotkeys Example

Create `~/.config/k9s/hotkeys.yaml`:

```yaml
hotKeys:
  shift-1:
    shortCut: Shift-1
    description: View pods
    command: pods
  shift-2:
    shortCut: Shift-2
    description: View deployments
    command: deploy
  shift-3:
    shortCut: Shift-3
    description: View services
    command: svc
```

### Custom Alias Example

Create `~/.config/k9s/aliases.yaml`:

```yaml
aliases:
  pp: v1/pods
  dd: apps/v1/deployments
  ss: v1/services
```

## Tips

| Tip | Details |
|-----|---------|
| `?` | List all keybindings (press `Esc` to exit help) |
| `:xray deploy` | Visualize deployment tree (pods, replicasets, etc.) |
| `:pulse` | Quick cluster health overview |
| `Ctrl+a` | Discover all available resource short names |
| `0` in any view | Toggle between current namespace and all namespaces |
| `Ctrl+s` | Save any resource YAML locally for backup |
| `:popeye` | Scan cluster for misconfigurations (requires Popeye) |
| Read-only mode | Use `k9s --readonly` for safe cluster browsing |
| Skins | Customize colors via skin YAML files in config directory |
| Plugins | Extend k9s with custom commands via `plugins.yaml` |

## Resources

* [https://github.com/derailed/k9s](https://github.com/derailed/k9s)
* [https://k9scli.io](https://k9scli.io)
* [https://k9scli.io/topics/commands](https://k9scli.io/topics/commands)
