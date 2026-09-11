# Git Credential Helpers: Storing and Retrieving Passwords and Tokens

A credential helper is how Git remembers the username/password or token it uses to talk to a remote over HTTPS, so you're not retyping it on every push. This guide covers checking which helper is active, the available helper types and their security tradeoffs, how the credential protocol works, and how to inspect or clear stored credentials.

## Check Which Helper Is Configured

```bash
# Show the global credential helper
git config --global credential.helper
```

Typical outputs:

| Output | Meaning |
|--------|---------|
| `osxkeychain` | Stored securely in the macOS Keychain |
| `manager` / `manager-core` | Windows Credential Manager (Git Credential Manager) |
| `libsecret` | Linux GNOME Keyring / Secret Service |
| `cache` | Held in memory temporarily, then forgotten |
| `store` | Written to a plaintext file (insecure) |
| *(no output)* | No helper configured — Git prompts every time |

To see the effective value **and where it came from** across all config scopes:

```bash
git config --show-origin --get-all credential.helper
```

That matters because helpers can be set at multiple levels (see below), and a repo-local or system config can override what you expect.

## How Credential Helpers Work

When you push or clone over HTTPS:

1. Git asks the configured helper for stored credentials matching the host.
2. If the helper returns them, Git uses them automatically — no prompt.
3. If nothing is stored, Git prompts you for username and password/token.
4. Git hands the newly entered credentials back to the helper, which stores them for next time.

The helper is invoked through a simple `get` / `store` / `erase` protocol keyed on protocol + host (+ optional path), which is why credentials are matched per remote host.

## The Helper Types

### macOS Keychain

```bash
git config --global credential.helper osxkeychain
```

Stores credentials in the macOS Keychain — encrypted and unlocked with your login. This is why they show up under Keychain Access and can be queried with `security find-internet-password`.

### Windows Credential Manager

```bash
git config --global credential.helper manager
```

Uses Git Credential Manager, backed by the Windows Credential Store. `manager-core` was the older name; modern Git for Windows ships `manager`.

### Linux (Secret Service / libsecret)

```bash
# Build/enable once, then:
git config --global credential.helper libsecret
```

Integrates with the GNOME Keyring or any Secret Service provider for encrypted storage on Linux.

### Cache (in-memory, temporary)

```bash
# Default 15-minute (900s) timeout
git config --global credential.helper cache

# Custom timeout — e.g. one hour
git config --global credential.helper 'cache --timeout=3600'
```

Keeps credentials in memory via a short-lived daemon, never touching disk. They're forgotten after the timeout or reboot. A good middle ground when there's no secure store available.

### Store (plaintext — use with caution)

```bash
git config --global credential.helper store

# Optional custom file location
git config --global credential.helper 'store --file=~/.git-credentials'
```

Writes credentials **unencrypted** to `~/.git-credentials` (mode 600, but still plaintext). Avoid on shared or untrusted machines; prefer a keychain or cache. Fine for headless/CI contexts where a token is scoped and disposable.

## Configuration Scopes

`credential.helper` can be set at three levels, most specific wins:

```bash
# System-wide (all users)
git config --system credential.helper <helper>

# Per-user (your account) — the usual choice
git config --global credential.helper <helper>

# Per-repository (this repo only)
git config --local credential.helper <helper>
```

To **disable** an inherited helper for a single repo (e.g. override a system default):

```bash
git config --local credential.helper ""
```

An empty value resets the list, so Git falls back to prompting.

## Per-Host Configuration

You can point different hosts at different helpers or usernames:

```bash
git config --global credential.https://github.com.helper manager
git config --global credential.https://github.com.username my-user
```

## Inspecting Stored Credentials

### macOS Keychain

```bash
# Find the stored GitHub credential
security find-internet-password -s github.com

# Include the secret value (prompts for keychain permission)
security find-internet-password -s github.com -w
```

Or open **Keychain Access** and search for the host.

### Plaintext store

```bash
# Credentials live here in the form https://user:token@host
cat ~/.git-credentials
```

### Ask Git directly via the protocol

```bash
printf 'protocol=https\nhost=github.com\n\n' | git credential fill
```

This runs the configured helper and prints what Git would use, without a network call.

## Clearing / Updating Credentials

When a token is rotated or wrong credentials got cached:

```bash
# Erase via the credential protocol (uses the active helper)
printf 'protocol=https\nhost=github.com\n\n' | git credential reject

# macOS: delete the keychain entry directly
security delete-internet-password -s github.com

# Plaintext store: edit or remove the file
rm ~/.git-credentials
```

The next push will prompt again and re-store the fresh credential.

## Recommendations

- Use the platform's secure store when available: **osxkeychain** (macOS), **manager** (Windows), **libsecret** (Linux).
- Prefer **cache** over **store** when no secure store exists — it keeps secrets off disk.
- Reserve **store** for disposable, scoped tokens (CI, headless boxes), never long-lived passwords on shared machines.
- On GitHub, use a Personal Access Token or the `gh` CLI (`gh auth login`) rather than an account password, which is no longer accepted for Git over HTTPS.
- Check `git config --show-origin --get-all credential.helper` when authentication behaves unexpectedly — an overriding scope is a common culprit.
