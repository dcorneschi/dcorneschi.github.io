# Installing Node.js on Ubuntu 22.04 and 24.04

There are several ways to get Node.js onto an Ubuntu 22.04 (Jammy) or 24.04 (Noble) machine, and the right one depends on whether you want the quickest setup, a specific/latest version, or the ability to juggle multiple versions per project. This guide covers five methods — the `apt` default repo, the NodeSource repository, `nvm`, `fnm`, and Docker — with when to use each, verification, and uninstall steps.

All commands assume a non-root user with `sudo` privileges. Node.js ships an npm bundle with every method below, so you rarely need to install npm separately.

## Which Method Should I Use?

| Method | Gets you | Best for |
|--------|----------|----------|
| `apt` default repo | Ubuntu's packaged version (older) | Quick start, no version requirements |
| NodeSource repo | Latest/specific major, system-wide | Servers needing a current, pinned major version |
| `nvm` | Any version, per-user, switchable | Developers juggling multiple Node versions |
| `fnm` | Any version, per-user, fast (Rust) | Same as nvm but faster shell startup |
| Docker | Isolated image, no host install | Containerized apps and CI |

Quick reference of what each Ubuntu LTS ships in its default repo (approximate — updated over the release's life):

| Ubuntu | Default-repo Node.js major |
|--------|----------------------------|
| 22.04 LTS (Jammy) | 12.x |
| 24.04 LTS (Noble) | 18.x |

## Method 1 — Default Ubuntu Repository (`apt`)

The fastest path. The packaged version lags behind current releases and mainly gets security fixes, so update it before using it in production.

```bash
sudo apt update
sudo apt install -y nodejs

# Optionally add npm (bundled separately in Ubuntu's packaging)
sudo apt install -y npm

# Verify
node -v
npm -v
```

Uninstall:

```bash
sudo apt remove nodejs          # remove package
sudo apt purge nodejs           # also remove config files
sudo apt autoremove             # clean up dependencies
```

Best when: you just need *a* Node.js and don't care about the exact version.

## Method 2 — NodeSource Repository (Latest/Specific Major, System-Wide)

[NodeSource](https://github.com/nodesource/distributions) maintains apt repositories for currently supported Node.js majors — more up to date than Ubuntu's. Pick the major you want in the setup URL (e.g. `setup_22.x` for Node 22 LTS, `setup_24.x` for Node 24).

```bash
# Download and inspect the setup script before running it
curl -fsSL https://deb.nodesource.com/setup_22.x -o nodesource_setup.sh
less nodesource_setup.sh

# Run it (configures the apt repo and GPG key)
sudo -E bash nodesource_setup.sh

# Install (npm is included automatically)
sudo apt install -y nodejs

# Verify
node -v
npm -v
```

Reviewing the script before piping it to a root shell is good practice for any `curl | bash`-style installer.

Uninstall (remove the package, then undo the repo the script added):

```bash
sudo apt remove -y nodejs
sudo rm -f /etc/apt/sources.list.d/nodesource.list
sudo rm -f /etc/apt/keyrings/nodesource.gpg
sudo apt update
```

Best when: a server needs a current, pinned major version installed system-wide via apt.

## Method 3 — Node Version Manager (`nvm`)

`nvm` installs Node.js per-user in your home directory and lets you install, list, and switch between multiple versions — ideal when different projects need different Node releases. It needs no `sudo` for Node installs.

```bash
# Install nvm (check github.com/nvm-sh/nvm for the current version tag)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.7/install.sh | bash

# Load nvm into the current shell (the installer also appends this to ~/.bashrc)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

Install and manage versions:

```bash
# List installable versions
nvm ls-remote

# Install a specific version
nvm install 22.6.0

# Install the latest LTS by codename or generically
nvm install --lts
nvm install lts/jod        # Node 22 LTS codename

# List what you have, set a default, switch versions
nvm ls
nvm alias default 22
nvm use 20
```

Uninstall a version (deactivate first if it's the active one):

```bash
nvm ls
nvm deactivate             # only if uninstalling the currently active version
nvm uninstall 20.11.1
```

Best when: you develop across projects that target different Node versions.

## Method 4 — Fast Node Manager (`fnm`)

`fnm` is a faster, Rust-based alternative to nvm with the same per-user, multi-version model and near-instant shell startup.

```bash
# Install fnm
curl -fsSL https://fnm.vercel.app/install | bash

# Reload your shell so fnm is on PATH
source ~/.bashrc
```

Install and manage versions:

```bash
# List available versions
fnm list-remote

# Install by version number (no leading "v")
fnm install 22.6.0

# Install latest LTS, set default, switch
fnm install --lts
fnm default 22
fnm list
fnm use 20
```

Uninstall a version — this works even if it's set as the default:

```bash
fnm uninstall 20.11.1
```

Best when: you want nvm-style version management but with faster shell initialization.

## Method 5 — Docker

If you're running containerized apps, skip installing Node.js on the host and use the official image. Pick a tag for the version and base (Alpine is smallest).

```bash
# Pull an image (substitute the major you want)
docker pull node:22-alpine

# Check versions inside the image
docker run --rm node:22-alpine node -v
docker run --rm node:22-alpine npm -v

# Run your app, mounting the project directory
docker run --rm -it -v "$PWD":/app -w /app node:22-alpine npm install
```

A minimal `Dockerfile` for a Node app:

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

Best when: you deploy in containers or want a clean, host-independent build in CI.

## Verifying the Install

Whatever method you used:

```bash
node -v      # e.g. v22.6.0
npm -v       # bundled npm version
which node   # where the binary resolved from (tells you which method is active)
```

If `node -v` shows an unexpected version, a version manager (`nvm`/`fnm`) on your `PATH` may be shadowing a system install — check `which node` and your shell rc files.

## Choosing a Version

- **LTS (even-numbered majors: 20, 22, 24)** — recommended for production; longer support windows. As of this writing, Node.js **24** is the Active LTS (supported through around April 2028) and **22** is in Maintenance LTS.
- **Current (odd-numbered majors)** — newest features, promoted from the six-month Current phase; odd majors never become LTS, so they suit experimentation, not production.

Node.js majors follow a predictable lifecycle: even majors go Current → Active LTS → Maintenance LTS (roughly 30 months of critical fixes total), while odd majors stay Current only. For production, install an LTS line, track the [Node.js release schedule](https://github.com/nodejs/release#release-schedule), and upgrade before your major reaches end-of-life. Read the changelog and test before rolling out a new major.

## References

- [NodeSource distributions](https://github.com/nodesource/distributions) — official apt/yum repos
- [nvm (nvm-sh)](https://github.com/nvm-sh/nvm) — Node Version Manager
- [fnm (Fast Node Manager)](https://github.com/Schniz/fnm) — Rust-based version manager
- [Node.js release schedule](https://github.com/nodejs/release#release-schedule) — LTS lifecycle
