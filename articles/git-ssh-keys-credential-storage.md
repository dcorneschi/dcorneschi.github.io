# Git SSH Keys and Credential Storage

Authenticating to a Git host comes down to two paths: **SSH keys** (key-based, no password prompts once set up) or **HTTPS credentials** (a token/password, optionally cached by a helper). This guide walks through generating and configuring an SSH key — including the port-443 workaround for firewalled networks — and the ways to store HTTPS credentials, with notes on the security tradeoffs of each.

## Generate an SSH Key

```bash
# Recommended: ed25519 (shorter, fast, secure)
ssh-keygen -t ed25519 -C "Daniel Corneschi"

# Alternative: RSA 4096-bit (use if a host lacks ed25519 support)
ssh-keygen -t rsa -b 4096 -C "Daniel Corneschi"
```

The `-C` comment is just a label (an email or name helps identify the key later). This creates a key pair — a private key (`id_ed25519`) and a public key (`id_ed25519.pub`). **Only the public key** gets uploaded to the Git host; the private key never leaves your machine.

To keep a dedicated key in its own directory (as in the original notes):

```bash
mkdir -p ~/git_key
ssh-keygen -t ed25519 -C "Daniel Corneschi" -f ~/git_key/id_ed25519
```

`-f` sets the output path. Ensure permissions are tight — SSH refuses keys that are too open:

```bash
chmod 700 ~/git_key
chmod 600 ~/git_key/id_ed25519
```

## Add the Public Key to Your Git Host

Copy the **public** key and paste it into your host's SSH-keys settings (GitHub: Settings → SSH and GPG keys):

```bash
cat ~/git_key/id_ed25519.pub
```

## Test the Connection

```bash
# Test using a specific key file
ssh -T git@github.com -i ~/git_key/id_ed25519
```

A successful test greets you by username (GitHub does not grant a shell, so a "does not provide shell access" message is expected and fine).

## SSH Config: Simplify and Work Around Firewalls

Rather than passing `-i` every time, describe the host in `~/.ssh/config`:

```bash
vi ~/.ssh/config
```

```text
Host github.com
    HostName ssh.github.com
    Port 443
    User git
    IdentityFile ~/git_key/id_ed25519
```

Two things this does:

- **`IdentityFile`** tells SSH which key to use for this host automatically.
- **`HostName ssh.github.com` + `Port 443`** routes SSH over the HTTPS port. Many corporate/university firewalls block port 22 (standard SSH) but allow 443 — this is GitHub's official workaround. On a normal network you can omit those two lines and use the default `Port 22`.

Test the config with:

```bash
ssh -T github.com
```

## Load the Key into an Agent (optional)

To avoid retyping the key passphrase each session:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/git_key/id_ed25519
```

## HTTPS Credential Storage

If you use HTTPS remotes instead of SSH, Git needs your username and a token. There are a few ways to persist them.

### Option 1: Credentials in the remote URL (not recommended)

```bash
git clone https://<USERNAME>:<TOKEN>@github.com/path/to/repo.git
```

The username and token get written **in plaintext** into `.git/config` as part of the remote URL. Convenient but insecure — anyone with the repo directory can read the token, and it can leak into logs. Prefer a credential helper below.

> On GitHub, use a Personal Access Token here, not your account password — password auth over HTTPS is no longer accepted.

### Option 2: The store credential helper

```bash
git config credential.helper store
```

On the next pull/push, Git prompts once for username and token, then saves them to `~/.git-credentials` (plaintext, file mode 600) and reuses them thereafter. Simple, but the token sits unencrypted on disk — acceptable for a personal machine or a scoped CI token, less so on shared systems.

### Option 3: Cache in memory (safer, temporary)

```bash
# Keep credentials in memory for the default 15 minutes
git config credential.helper cache

# Custom timeout (seconds)
git config credential.helper 'cache --timeout=3600'
```

Nothing hits the disk; credentials are forgotten after the timeout.

### Option 4: OS keychain (most secure)

Use the platform's encrypted store where available — `osxkeychain` (macOS), `manager` (Windows), `libsecret` (Linux). See [Git Credential Helpers](articles/git-credential-helpers.md) for the full breakdown of helper types, config scopes, and inspecting or clearing stored credentials.

## SSH vs HTTPS — Which to Use

| | SSH | HTTPS |
|---|-----|-------|
| Setup | Generate + upload a key once | None, or a token |
| Prompts | None after setup | Depends on helper |
| Firewall friendliness | Port 22 often blocked (use 443 workaround) | Works through most proxies |
| Best for | Frequent contributors, automation | Quick clones, restricted networks |

## Summary

- Generate an `ed25519` key (`ssh-keygen -t ed25519`), upload the `.pub` half, and test with `ssh -T git@github.com`.
- Use `~/.ssh/config` to auto-select the key; add `HostName ssh.github.com` + `Port 443` when port 22 is blocked.
- For HTTPS, avoid putting the token in the clone URL; prefer a credential helper — `cache` (memory) or the OS keychain over plaintext `store`.
- On GitHub, HTTPS auth needs a Personal Access Token, not a password.
