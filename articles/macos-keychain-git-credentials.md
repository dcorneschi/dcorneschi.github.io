# Finding and Managing Git Credentials in the macOS Keychain

When you clone or push over HTTPS on macOS, Git stores the password or personal access token (PAT) in the login Keychain via the `osxkeychain` credential helper. This guide shows how to find those stored credentials — through the Keychain Access app, the `security` command line, and Git's own credential helper — plus how to update or remove them when a token rotates.

## Using Keychain Access (GUI)

1. Open **Keychain Access** (Applications → Utilities → Keychain Access).
2. In the search bar, type your Git service domain — `github.com`, `gitlab.com`, or your Git server's URL.
3. Look for entries of kind **Internet password**.
4. Double-click the entry, then check **Show password**.
5. Enter your macOS login password when prompted to reveal it.

## Using the Command Line (`security`)

The `security` tool queries the Keychain directly.

```bash
# Show only the password/token for a host
security find-internet-password -s github.com -w
security find-internet-password -s gitlab.com -w
security find-internet-password -s your-git-server.com -w

# Show full entry details (account, where, dates) instead of just the password
security find-internet-password -s github.com

# Narrow to a specific account when multiple entries exist for one host
security find-internet-password -s github.com -a your-username -w
```

The `-w` flag prints just the secret. Omit it to see the metadata (account, protocol, timestamps).

> Be careful printing secrets: `-w` writes the token to your terminal and shell history/scrollback. Pipe it straight to the clipboard instead of displaying it:
>
> ```bash
> security find-internet-password -s github.com -w | tr -d '\n' | pbcopy
> ```

## Git's Credential Helper

On macOS, Git talks to the Keychain through the `osxkeychain` helper.

```bash
# See which helper Git is configured to use (expect: osxkeychain)
git config --global credential.helper

# Ask the helper what it has stored for a host (interactive)
git credential fill
# then type, ending with a blank line:
#   protocol=https
#   host=github.com
#   <blank line>
```

If `credential.helper` returns nothing, enable the built-in macOS helper:

```bash
git config --global credential.helper osxkeychain
```

## What the Stored Entry Looks Like

Git credentials are saved as **Internet password** items:

| Field | Value |
|-------|-------|
| Account | Your username or email (for PATs, often the username or `x-access-token`) |
| Where | The Git server URL — `https://github.com`, `https://gitlab.com`, etc. |
| Kind | Internet password |

Entry names typically look like `https://github.com`, `https://gitlab.com`, or your custom server URL.

## Updating or Removing a Credential

When a PAT expires or you rotate it, the old cached credential causes auth failures. Clear it so Git prompts again.

### Via the command line

```bash
# Delete the stored entry for a host (Git will re-prompt on next use)
security delete-internet-password -s github.com

# Delete a specific account's entry on that host
security delete-internet-password -s github.com -a your-username
```

### Via Git's helper

```bash
# Erase what the helper has for a host (interactive)
git credential reject
#   protocol=https
#   host=github.com
#   <blank line>
```

### Via Keychain Access (GUI)

Search for the host, right-click the Internet password entry, and choose **Delete**.

After removing it, the next `git pull`/`push` over HTTPS prompts for credentials, and the helper stores the new value.

## Storing a New Credential Non-Interactively

You can seed a credential directly (useful in setup scripts). For GitHub and most services, use a **personal access token** as the password, not your account password.

```bash
git credential approve <<EOF
protocol=https
host=github.com
username=your-username
password=ghp_your_personal_access_token
EOF
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| No Git entries in Keychain | Using SSH keys, or helper not enabled | Check `git config --global credential.helper`; SSH auth doesn't use these |
| Auth fails after rotating a token | Old token still cached | Delete the entry (`security delete-internet-password -s <host>`) and retry |
| `git` keeps prompting every time | No credential helper configured | `git config --global credential.helper osxkeychain` |
| Wrong account used for a host | Multiple entries for the same host | Delete the stale one, or target with `-a <account>` |
| `find-internet-password` returns nothing | Host string mismatch | Match the stored "Where" exactly (bare host like `github.com`, not the full URL) |

## Security Notes

- Treat retrieved tokens like passwords — avoid echoing them to the terminal; pipe to `pbcopy` instead.
- Prefer **fine-grained or scoped PATs** over broad tokens, and rotate them periodically.
- For frequent multi-account or key-based workflows, consider SSH keys instead of HTTPS+PAT — see the related SSH articles below.

## Related

- [SSH Managing Multiple Keys](articles/ssh-managing-multiple-keys.md) — per-service SSH keys for GitHub, GitLab, and homelab hosts
- [VS Code Git Actions and Git CLI Equivalents](articles/vscode-git-cli-equivalents.md) — mapping editor Source Control actions to git commands
