# Git Clone Methods and Options

`git clone` copies a repository, but the protocol you use, the depth of history you pull, and the options you pass all shape the result. This guide covers the transport methods (HTTPS, SSH, local, and the deprecated git protocol), authentication, and the everyday options — shallow clones, single branch, submodules, custom directories, and mirrors.

## Transport Methods

### HTTPS (most common)

```bash
git clone https://github.com/username/repository.git
```

Works through firewalls and proxies with no SSH setup — the easiest starting point. It may prompt for a token/credentials on private repos and is marginally slower than SSH.

### SSH (best for frequent contributors)

```bash
git clone git@github.com:username/repository.git
```

Once your SSH key is configured, there are no password prompts and authentication is key-based. It can be blocked by restrictive firewalls, and it requires one-time key setup.

### Local path

```bash
git clone /path/to/repository
git clone file:///path/to/repository
```

Clone from the local filesystem — handy for backups, test copies, or network shares. A plain path may hardlink objects for speed; the `file://` form forces a normal copy.

### git:// protocol (deprecated)

```bash
git clone git://github.com/username/repository.git
```

Read-only, unauthenticated, and largely retired — GitHub and most hosts no longer serve it. Avoid for anything new.

## Common Clone Options

### Shallow clone (limited history)

```bash
# Only the latest commit
git clone --depth 1 https://github.com/username/repository.git
```

Faster and smaller — ideal for CI/CD where full history isn't needed. Deepen later with `git fetch --deepen <n>` or `git fetch --unshallow` for the complete history.

### A specific branch

```bash
git clone -b branch-name https://github.com/username/repository.git

# Fetch ONLY that branch's history
git clone -b branch-name --single-branch https://github.com/username/repository.git
```

`-b` also accepts a tag to check out. `--single-branch` skips fetching all other branches.

### With submodules

```bash
# Clone and initialize submodules in one step
git clone --recurse-submodules https://github.com/username/repository.git

# --recursive is the older alias
git clone --recursive https://github.com/username/repository.git

# Fetch submodules in parallel for speed
git clone --recurse-submodules -j8 https://github.com/username/repository.git
```

If you forget, initialize afterward with `git submodule update --init --recursive`.

### Choosing the target directory

```bash
# Clone into a custom directory name
git clone https://github.com/username/original-name.git my-custom-name

# Clone into the current directory
git clone https://github.com/username/repo.git .

# Example: LazyVim into the Neovim config path
git clone https://github.com/LazyVim/starter ~/.config/nvim
```

### Bare and mirror clones

```bash
# No working tree — just the .git database (for servers/backups)
git clone --bare https://github.com/username/repo.git

# Full mirror: all refs, set up to be re-pushed as a mirror
git clone --mirror https://github.com/username/repo.git
```

Use `--mirror` when backing up or relocating a repository with every branch, tag, and remote ref intact.

## Authentication

### Personal Access Tokens (HTTPS)

```bash
# Embed a token (avoid — it lands in shell history and remotes)
git clone https://<token>@github.com/username/repository.git
```

Prefer letting a credential helper supply the token so it isn't baked into the URL. GitHub no longer accepts account passwords over HTTPS — use a PAT or the `gh` CLI.

**GitLab** uses an `oauth2:` username prefix with a Personal Access Token (minimum scope `read_repository` for clone):

```bash
git clone https://oauth2:<TOKEN>@gitlab.com/group/repo.git
```

The same form works against a self-hosted instance (`https://oauth2:<TOKEN>@gitlab.example.com/...`). As with GitHub, prefer a credential helper over embedding the token in the URL.

### SSH keys

```bash
# Generate a key, then add the public half to your Git host
ssh-keygen -t ed25519 -C "your_email@example.com"

# Verify the connection
ssh -T git@github.com
```

### Credential helpers

```bash
# Cache in memory temporarily
git config --global credential.helper cache

# Persist to disk (plaintext — use a keychain where possible)
git config --global credential.helper store
```

For the full breakdown of helper types and their security tradeoffs, see [Git Credential Helpers](articles/git-credential-helpers.md).

## Practical Examples

```bash
# SSH clone
git clone git@github.com:facebook/react.git

# Clone a specific branch
git clone -b develop https://github.com/username/project.git

# Shallow, single-branch clone for CI/CD
git clone --depth 1 --single-branch https://github.com/username/large-repo.git

# Clone with submodules, parallelized
git clone --recurse-submodules -j8 https://github.com/username/project.git

# Mirror clone for backup
git clone --mirror https://github.com/username/repo.git
```

## Best Practices

- Use **SSH** for repos you contribute to regularly; **HTTPS** for quick or public clones.
- Reach for **`--depth 1 --single-branch`** on large repos or in CI when history isn't needed.
- Keep tokens out of clone URLs — let a credential helper provide them.
- Verify the repository URL before cloning to avoid typosquatted or malicious sources.
- Add **`--recurse-submodules`** up front for projects that use submodules to save a follow-up step.
