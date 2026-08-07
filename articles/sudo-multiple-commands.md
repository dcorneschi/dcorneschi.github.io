# Running Multiple Commands with sudo

`sudo` elevates a single command to run with root (or another user's) privileges. When you need to run several commands as root, there are a few approaches — each with different trade-offs for readability, safety, and scope.

## The Problem

Running commands one at a time gets repetitive:

```bash
sudo apt update
sudo apt install nginx
sudo systemctl enable nginx
```

There are cleaner ways to handle this.

## Approach 1 — Semicolons or `&&` Inside a Subshell

```bash
sudo bash -c "apt update; apt install -y nginx; systemctl enable nginx"
```

Using `&&` instead stops execution on failure:

```bash
sudo bash -c "apt update && apt install -y nginx && systemctl enable nginx"
```

`bash -c` launches a subshell as root and runs the entire string inside it. All commands inherit root privileges for the lifetime of that subshell.

> **Note:** The quoting matters. Wrap the entire command string in double quotes. If any individual command itself uses double quotes, escape them (`\"`) or switch to single quotes for the outer wrapper.

## Approach 2 — Heredoc (Multiple Lines, Readable)

```bash
sudo bash <<'EOF'
apt update
apt install -y nginx
systemctl enable nginx
systemctl start nginx
EOF
```

The `<<'EOF'` (quoted heredoc) prevents variable expansion in the block, so `$variables` are passed literally to the subshell. Use `<<EOF` (unquoted) if you want variables to expand in the calling shell before being handed to sudo.

```bash
PACKAGE="nginx"

# Variables expand before sudo sees them
sudo bash <<EOF
apt install -y $PACKAGE
systemctl enable $PACKAGE
EOF
```

## Approach 3 — `sudo -s` (Interactive Root Shell)

```bash
sudo -s
```

Starts an interactive root shell using the `SHELL` environment variable. You run multiple commands manually and exit when done.

```bash
sudo -s
# Now running as root
apt update
apt install -y nginx
exit
```

> **Note:** `sudo -s` does not reset the environment. Use `sudo -i` if you need a full login shell with root's `PATH`, `HOME`, and profile scripts sourced.

## Approach 4 — `sudo -i` (Full Login Shell)

```bash
sudo -i
```

Starts a login shell as root, sourcing `/root/.profile` and `/root/.bashrc`. This is the closest equivalent to logging in directly as root.

Useful when environment variables like `PATH` or `HOME` pointing to root's directories are needed for the commands to work correctly.

## Approach 5 — Script File

For anything beyond a few commands, a script is cleaner:

```bash
#!/bin/bash
set -euo pipefail

apt update
apt install -y nginx
systemctl enable nginx
systemctl start nginx
```

```bash
sudo bash setup.sh
```

Or make the script executable and call sudo directly:

```bash
chmod +x setup.sh
sudo ./setup.sh
```

> **Note:** `sudo` looks at the file's owner, not the caller. The script executes as root because you prefixed it with `sudo`, not because of the script's own permissions.

## Approach 6 — Run as a Specific User

`sudo` is not limited to root. Use `-u` to run multiple commands as any user on the system:

```bash
sudo -u postgres -- sh -c 'pg_dumpall > /tmp/backup.sql && gzip /tmp/backup.sql'
sudo -u mysql -- sh -c '/home/mysql/backup.sh; /home/mysql/mirror.py'
```

The `--` signals the end of sudo's own option parsing, so anything after it is passed directly to the shell. This avoids ambiguity when the command string starts with something that looks like a flag.

You can use `--` with root as well:

```bash
sudo -- sh -c 'apt update && apt upgrade'
```

## Approach 7 — `sudo su`

```bash
sudo su
```

Switches to the root user by running `su` under sudo. Functionally similar to `sudo -i` but slightly different: `su` reads root's login shell from `/etc/passwd`, while `sudo -i` uses the `SHELL` variable or the sudoers-configured shell.

For scripting, prefer `sudo bash -c` or a script file. `sudo su` is more of an interactive convenience.

## Comparison

| Approach | Scope | Best for |
|----------|-------|----------|
| `sudo bash -c "..."` | Subshell | Short, inline multi-command sequences |
| `sudo bash <<'EOF'` | Subshell | Multi-line blocks without a separate file |
| `sudo -s` | Interactive shell | Quick interactive work as root |
| `sudo -i` | Login shell | Root work that depends on root's environment |
| `sudo bash script.sh` | Script file | Repeatable or complex automation |
| `sudo -u user -- sh -c '...'` | Subshell | Running as a specific non-root user |
| `sudo su` | Login shell | Interactive; equivalent to `sudo -i` in most cases |

## Pipes and Redirection

Pipes and redirections are interpreted by the shell before `sudo` runs, so the elevated privileges don't apply to them. This is a common source of unexpected "permission denied" errors.

### Piping output to a root-owned file

```bash
# Fails — the >> redirection runs as your user, not root
sudo echo "option=value" >> /etc/app/config.conf

# Works — tee runs as root and writes the file
echo "option=value" | sudo tee -a /etc/app/config.conf > /dev/null
```

`tee -a` appends to the file. The `> /dev/null` suppresses tee's stdout echo. Use `tee` (without `-a`) to overwrite instead of append.

### Pipe inside a root shell

Wrapping everything in a subshell keeps both the command and the redirection under root:

```bash
sudo sh -c 'apt-get update && apt-get upgrade | tee /var/log/upgrade.log'
```

Here `tee` can write to the root-owned log file because the entire pipeline runs inside the elevated shell.

## Logical Operators

You're not limited to `&&` and `;`. The full set of shell logical operators works inside a subshell:

| Operator | Behaviour |
|----------|-----------|
| `;` | Run next command regardless of exit code |
| `&&` | Run next command only if previous succeeded (exit 0) |
| `\|\|` | Run next command only if previous failed (non-zero exit) |

```bash
# Stop on failure
sudo sh -c 'mkdir /backup && cp -r /home/user/* /backup'

# Fallback: try install, update repos if it fails
sudo sh -c 'apt-get install foo || apt-get update'

# Mix: update, then upgrade or log failure
sudo sh -c 'apt-get update && apt-get upgrade || echo "upgrade failed" >> /var/log/upgrade-errors.log'
```

You can also combine both operators in a single expression. The classic pattern is create-then-write-or-cleanup:

```bash
sudo sh -c 'touch test.txt && echo "hello" > test.txt || rm test.txt'
```

- If `touch` succeeds, `echo` runs to write the content
- If `touch` or `echo` fails, `rm` cleans up the partial file

## Running Commands in Parallel

Use `&` inside the subshell to background tasks and `wait` to block until they all finish:

```bash
sudo sh -c 'task1 & task2 & wait'
```

Useful for independent long-running operations, like running two backup jobs at the same time:

```bash
sudo sh -c '/opt/backup/db.sh & /opt/backup/files.sh & wait && echo "all done"'
```

> **Note:** Output from parallel tasks will interleave. Redirect each command to its own log file if the output matters.

## Preserving Environment Variables

By default, `sudo` strips most environment variables before running the command (controlled by `env_reset` in `/etc/sudoers`). If a command depends on variables set in your current shell, pass them explicitly:

```bash
# Pass a single variable
sudo MY_VAR="value" bash -c 'echo $MY_VAR'

# Preserve specific variables using -E flag (if allowed by sudoers)
sudo -E bash -c 'echo $MY_VAR'
```

> **Note:** `sudo -E` only works if the sudoers policy permits it (`!env_reset` or `env_keep`). Many distributions restrict this by default.

## Key Points

- `sudo bash -c "..."` is the quickest way to run multiple commands without opening a shell
- Use `&&` instead of `;` if you want the sequence to stop on failure
- Use `||` for fallback logic — the right side runs only when the left side fails
- Heredocs (`<<'EOF'`) are easier to read than long one-liners
- `sudo -i` gives a full root login shell; `sudo -s` keeps your current environment
- For anything more than a few commands, write a script and `sudo bash script.sh`
- Pipes and redirections run as your user, not root — use `tee` or wrap in a subshell
- Use `sudo -u username -- sh -c '...'` to run multiple commands as a specific non-root user
- `--` stops sudo from parsing its own flags; use it when the command string could be misread
