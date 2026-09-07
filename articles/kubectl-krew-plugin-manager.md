# Krew: The kubectl Plugin Manager

[Krew](https://krew.sigs.k8s.io) is the plugin manager for `kubectl`. It lets
you discover, install, and update kubectl plugins from a curated index, much
like `apt` or `brew` do for their ecosystems.

## Install Krew

Krew is itself installed as a kubectl plugin. On macOS/Linux:

```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```

Add krew to your `PATH` (add this line to `~/.bashrc` or `~/.zshrc`):

```bash
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

Verify:

```bash
kubectl krew version
```

## Managing Plugins

```bash
# Install a plugin
kubectl krew install <plugin-name>

# List installed plugins
kubectl krew list

# Show info about a plugin
kubectl krew info <plugin-name>

# Upgrade all installed plugins
kubectl krew upgrade

# Upgrade a single plugin
kubectl krew upgrade <plugin-name>

# Uninstall a plugin
kubectl krew uninstall <plugin-name>
```

Once installed, a plugin is invoked as a kubectl subcommand — e.g. installing
`ctx` gives you `kubectl ctx`.

## Searching for Plugins

### List all available plugins (default index)

```bash
kubectl krew search
```

### Search for a specific plugin

```bash
kubectl krew search <plugin-name>
```

## Working with Indexes

Krew reads plugins from one or more *indexes* (git repositories of plugin
manifests). The default index, added automatically on install, is the official
[krew-index](https://github.com/kubernetes-sigs/krew-index).

### List all configured indexes

```bash
kubectl krew index list
```

### Add a custom index

You can add third-party indexes to install plugins not in the official one:

```bash
# Format: krew index add <name> <git-repo-url>
kubectl krew index add myindex https://github.com/example/my-krew-index
```

### Search a specific index

```bash
kubectl krew search --index=myindex
```

### Install from a specific index

Reference the plugin as `<index>/<plugin>`:

```bash
kubectl krew install myindex/<plugin-name>
```

### Remove an index

```bash
kubectl krew index remove myindex
```

> **Note:** Krew's primary source is the official `krew-index` on GitHub. For
> most use cases the default index via `kubectl krew search` is enough to find
> the commonly used kubectl plugins. (Helm Hub / Artifact Hub is a separate
> project for Helm charts and is not a krew index.)

## Popular Plugins

- `ctx` / `ns` — fast context and namespace switching
- `kube-capacity` — node/pod resource requests, limits, and utilization ([repo](https://github.com/robscott/kube-capacity))
- `neat` — strip clutter from `kubectl get -o yaml` output
- `tree` — show ownership hierarchy of resources
- `access-matrix` — RBAC access review for a resource

## Resources

- [Krew documentation](https://krew.sigs.k8s.io)
- [Official krew-index](https://github.com/kubernetes-sigs/krew-index)
- [kube-capacity plugin](https://github.com/robscott/kube-capacity)

## Skills Practiced

- Installing krew and adding it to `PATH`
- Installing, listing, upgrading, and removing kubectl plugins
- Searching the default index and adding/searching custom indexes
- Installing plugins from a specific index with `<index>/<plugin>` syntax
