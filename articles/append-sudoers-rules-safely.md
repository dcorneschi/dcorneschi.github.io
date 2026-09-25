# Appending Rules to sudoers.d Files Safely

Dropping a file into `/etc/sudoers.d/` is the standard way to grant extra sudo rights without touching the main `/etc/sudoers`. A common need in provisioning scripts (cloud-init, user-data, Ansible `raw`) is to **append** a blank separator line plus one or more rules to such a file. This guide shows the ways to do that, and — more importantly — how to do it safely, because a malformed sudoers file can lock everyone out of `sudo`.

For the syntax of sudoers rules themselves, see the [sudoers Guide](articles/sudo-sudoers-guide.md); for restricting a user to specific commands, [Running Multiple Commands with sudo](articles/sudo-multiple-commands.md).

> **Two rules that are not optional.** Every file in `/etc/sudoers.d/` must be mode `0440` (sudo ignores group/world-writable files), and you must **validate** with `visudo -c` before trusting it. An invalid drop-in can break `sudo` entirely — see [Validate Before You Commit](#validate-before-you-commit).

## Quick Ways to Append (Interactive / Trusted Input)

These one-liners prepend a blank line (`\n`) so the new rules are visually separated from existing content. Use them when you're confident in the input and will validate afterward.

### echo -e

```bash
echo -e "\nline1\nline2" | sudo tee -a /etc/sudoers.d/myfile
```

### printf (most portable)

`printf` behaves consistently across shells, whereas `echo -e` is not portable (some shells print the literal `-e`). Prefer `printf` in scripts:

```bash
printf '\nline1\nline2\n' | sudo tee -a /etc/sudoers.d/myfile
```

### Separate echo commands

```bash
{ echo ""; echo "line1"; echo "line2"; } | sudo tee -a /etc/sudoers.d/myfile
```

### Heredoc (cleanest for many lines)

Quote the delimiter (`'EOF'`) so nothing inside is expanded by the shell — important for sudoers, which contains no shell variables:

```bash
sudo tee -a /etc/sudoers.d/myfile << 'EOF'

# Docker permissions
ubuntu ALL=(ALL) NOPASSWD: /usr/bin/docker
ubuntu ALL=(ALL) NOPASSWD: /usr/bin/systemctl
EOF
```

A real sudoers example with `tee -a`:

```bash
echo -e "\nubuntu ALL=(ALL) NOPASSWD: /usr/bin/docker\nubuntu ALL=(ALL) NOPASSWD: /usr/bin/systemctl" \
  | sudo tee -a /etc/sudoers.d/ubuntu
sudo chmod 0440 /etc/sudoers.d/ubuntu
```

> `tee -a` **appends**; plain `tee` (or `>`) **overwrites**. Use `-a` when adding to an existing file. Note that redirection like `sudo command > file` runs the redirect as *your* user, not root — piping to `sudo tee` is the correct way to write a root-owned file.

## Validate Before You Commit

The safe pattern is: build the content in a **temp file**, validate it with `visudo -c -f`, and only then append it to the real drop-in. If validation fails, nothing touches `/etc/sudoers.d/` and `sudo` stays working.

```bash
#!/bin/bash
set -euo pipefail

tmp=$(mktemp)
printf '\n# Service account permissions\nubuntu ALL=(ALL) NOPASSWD: /usr/bin/docker\nubuntu ALL=(ALL) NOPASSWD: /usr/bin/systemctl\n' > "$tmp"

if visudo -c -f "$tmp" >/dev/null 2>&1; then
    cat "$tmp" >> /etc/sudoers.d/custom
    chmod 0440 /etc/sudoers.d/custom
    echo "Rules added successfully"
else
    echo "ERROR: invalid sudoers syntax, not applied" >&2
    rm -f "$tmp"
    exit 1
fi

rm -f "$tmp"
```

`visudo -c -f <file>` checks a specific file for syntax errors without opening an editor. It's the same parser sudo uses at runtime, so a passing check means the file is safe to install.

### Validate a drop-in that's already in place

```bash
sudo visudo -c -f /etc/sudoers.d/custom   # check one file
sudo visudo -c                            # check the whole sudoers set
```

### Editing an existing drop-in interactively

To *edit* (not append) a drop-in with automatic validation on save, point `visudo` at it:

```bash
sudo visudo -f /etc/sudoers.d/custom
```

`visudo` refuses to save a file with syntax errors, which is why it's preferred over a plain editor for anything under `/etc/sudoers.d/`.

## A Reusable Function

For scripts that add rules in several places, wrap the validate-then-apply pattern:

```bash
add_sudoers_rules() {
    local file="$1"; shift          # first arg = drop-in name; rest = rules
    local tmp; tmp=$(mktemp)

    printf '\n' > "$tmp"            # leading blank separator line
    local rule
    for rule in "$@"; do
        printf '%s\n' "$rule" >> "$tmp"
    done

    if visudo -c -f "$tmp" >/dev/null 2>&1; then
        cat "$tmp" >> "/etc/sudoers.d/$file"
        chmod 0440 "/etc/sudoers.d/$file"
        rm -f "$tmp"
        echo "Rules added to /etc/sudoers.d/$file"
    else
        echo "Validation failed; no changes made" >&2
        rm -f "$tmp"
        return 1
    fi
}

# Usage
add_sudoers_rules myfile \
    "ubuntu ALL=(ALL) NOPASSWD: /usr/bin/docker" \
    "ubuntu ALL=(ALL) NOPASSWD: /usr/bin/systemctl"
```

Building the temp file with a loop and `printf '%s\n'` avoids the `echo -e "$content"` approach, which mangles rules that happen to contain backslash sequences.

## Naming and Content Rules for sudoers.d

A few constraints that cause silent failures if ignored:

- **File mode must be `0440`** (root-owned, read-only). sudo skips files that are group- or world-writable.
- **Filenames must not contain a `.` or end with `~`.** sudo ignores drop-ins with a dot in the name (e.g. `myrules.conf` is skipped) and editor backup files ending in `~`.
- **One logical rule per line.** End the file with a trailing newline.
- Files are read in **lexical order**; later files can override earlier ones.

## Key Takeaways

- Use `sudo tee -a` to append to a `/etc/sudoers.d/` file; `printf '\n...'` is the portable way to add a leading blank line and rules.
- **Always validate** with `visudo -c -f` before installing, ideally against a temp file so a bad rule never reaches the live directory.
- **Always `chmod 0440`** the drop-in, and avoid `.`/`~` in the filename or sudo will ignore it.
- Prefer heredocs with a quoted delimiter (`<< 'EOF'`) for multiple lines; prefer `printf`/loops over `echo -e` in scripts.
- Piping to `sudo tee` writes the root-owned file correctly; a plain `sudo cmd > file` redirect does not.

## Quick Reference

```bash
# Append a blank line + rules (validate afterward)
printf '\n%s\n%s\n' \
  "ubuntu ALL=(ALL) NOPASSWD: /usr/bin/docker" \
  "ubuntu ALL=(ALL) NOPASSWD: /usr/bin/systemctl" | sudo tee -a /etc/sudoers.d/ubuntu
sudo chmod 0440 /etc/sudoers.d/ubuntu
sudo visudo -c -f /etc/sudoers.d/ubuntu

# Safe pattern: validate a temp file, then install
tmp=$(mktemp)
printf '\nubuntu ALL=(ALL) NOPASSWD: /usr/bin/docker\n' > "$tmp"
visudo -c -f "$tmp" && install -m 0440 "$tmp" /etc/sudoers.d/ubuntu; rm -f "$tmp"
```

For related material, see the [sudoers Guide](articles/sudo-sudoers-guide.md) and [Running Multiple Commands with sudo](articles/sudo-multiple-commands.md).
