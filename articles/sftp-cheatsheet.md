# SFTP Cheatsheet

`sftp` is an interactive file-transfer client that runs over SSH, so you get the same authentication, encryption, and host keys as an SSH login — no separate service or credentials. It presents a shell-like prompt where **remote** commands (`ls`, `cd`, `get`, `put`) have **local** counterparts prefixed with `l` (`lls`, `lcd`, `lpwd`). This cheatsheet covers connecting, navigating, transferring, and scripting.

For server-side restriction of SFTP users, see [Chroot SFTP Setup](articles/chroot-sftp-setup.md); for syncing to cloud/object storage, the [rclone Cheatsheet](articles/rclone-cheatsheet.md).

## Connecting

```sh
sftp user@host                 # connect (SSH defaults apply)
sftp -P 2222 user@host         # non-standard SSH port (capital -P)
sftp -i ~/.ssh/id_ed25519 user@host   # connect with a specific key
sftp -b commands.txt user@host # batch mode: run commands from a file
```

| Flag | Meaning |
|------|---------|
| `-P port` | Connect on a non-standard SSH port (note: **capital** P) |
| `-i keyfile` | Use a specific private key |
| `-b batchfile` | Read commands from a file (non-interactive); `-` reads stdin |
| `-r` | Recurse (used with `get`/`put`, not at connect time) |

> `sftp` uses **`-P`** (uppercase) for the port, unlike `ssh`/`scp` which use lowercase `-p`. Also unlike `scp`, `sftp -P` matches `scp -P` — but `ssh` uses `-p`. Mixing these up is a common gotcha.

Close the session with `quit`, `exit`, or `bye`.

## Navigation

Remote commands act on the server; the `l`-prefixed versions act on your local machine.

| Remote | Local | Description |
|--------|-------|-------------|
| `pwd` | `lpwd` | Print working directory |
| `cd path` | `lcd path` | Change directory |
| `ls` / `ls -la` | `lls` / `lls -la` | List files |
| `mkdir dir` | `lmkdir dir` | Create directory |

## Transferring Files

### Download: remote → local (get)

```sh
get file                       # download one file to the local cwd
get file newname               # download and rename
get -r directory               # download a directory recursively
mget *.txt                     # download multiple files by glob
```

### Upload: local → remote (put)

```sh
put file                       # upload one file to the remote cwd
put file newname               # upload and rename
put -r directory               # upload a directory recursively
mput *.txt                     # upload multiple files by glob
```

`get`/`put` also accept `-p` (lowercase) to **preserve** modification times and permissions:

```sh
get -p report.pdf              # keep timestamps/permissions on download
put -pr ./site                 # recursive upload, preserving attributes
```

## File Management (remote)

| Command | Description |
|---------|-------------|
| `rm file` | Delete a file |
| `rmdir directory` | Delete an empty directory |
| `mkdir directory` | Create a directory |
| `rename old new` | Rename/move a file |
| `chmod 644 file` | Change permissions |
| `chown user file` | Change owner (numeric UID on many servers) |
| `chgrp group file` | Change group |
| `ln -s target link` / `symlink target link` | Create a symbolic link |
| `readlink link` | Show a symlink's target |

> On many servers `chown`/`chgrp` expect **numeric** UID/GID rather than names, because the SFTP subsystem may not resolve names the same way the shell does. If `chown user:group file` fails, try the numeric IDs.

## Information and Help

| Command | Description |
|---------|-------------|
| `help` / `?` | List available commands |
| `help command` | Help for a specific command |
| `stat file` | File info (remote) |
| `lstat file` | File info (local) |
| `df` / `df -h` | Remote filesystem usage |
| `du path` | Disk usage of a remote path |
| `version` | Show the SFTP protocol version |
| `!command` | Run `command` in a local shell |
| `!` | Drop to a local shell (exit to return) |

The `!` escape is handy: `!ls -la` runs a local `ls` without leaving the SFTP session.

## Batch and Non-Interactive Use

For automation, feed commands from a file (or stdin) with `-b`. Combine with key auth so nothing prompts:

```sh
# commands.txt
cd /var/www/html
put -r ./build
chmod 644 index.html
bye
```

```sh
sftp -i ~/.ssh/deploy_key -b commands.txt user@host
```

In batch mode, any failing command aborts the session by default. Prefix a command with `-` to ignore its error and continue (e.g. `-rm oldfile`).

> For scripted transfers, SSH keys avoid interactive password prompts. Never hard-code passwords; use a key (optionally with an agent) instead.

## Common Workflows

```sh
# Upload an entire project
sftp user@host
cd /path/to/destination
put -r ./project
bye
```

```sh
# Download and rename a file
sftp user@host
get server_config.json local_config.json
bye
```

```sh
# Pull all logs to the local machine
sftp user@host
cd /var/log
lcd ~/host-logs
mget *.log
bye
```

## Tips

- **Local vs remote:** remote commands have `l`-prefixed local twins (`ls`/`lls`, `cd`/`lcd`, `pwd`/`lpwd`).
- **Set both directories:** use `cd` (remote) and `lcd` (local) before `mget`/`mput` so files land where you want.
- **Glob patterns:** `*` and `?` work with `mget`/`mput`; if the local shell interferes, quote them: `mget "*.txt"`.
- **Paths with spaces:** quote them — `cd "My Documents"`.
- **Tab completion:** modern `sftp` completes both remote and local paths.
- **Relative paths:** `.` = current dir, `..` = parent.
- **When to reach for something else:** `sftp` has no delta sync. For large repeat transfers or mirroring, `rsync` (over SSH) or [rclone](articles/rclone-cheatsheet.md) is far more efficient.

## Common Issues

| Issue | Solution |
|-------|----------|
| Connection refused | Verify host, port (`-P`), credentials, and firewall/SSH service |
| Permission denied | Check remote file/dir permissions and the user's privileges |
| `chown`/`chmod` fails | Server may need numeric UID/GID, or the user lacks rights |
| Glob not matching | Quote the pattern: `mget "*.txt"` |
| Slow transfers | Latency-bound; consider `rsync`/rclone, or enable SSH compression |
| Wrong port error | `sftp` uses **`-P`** (uppercase), not `-p` |

## Quick Reference

```sh
sftp -P 2222 -i ~/.ssh/id_ed25519 user@host   # connect (port + key)
cd /remote/dir     ;  lcd /local/dir           # set remote & local dirs
get -r dir         ;  put -r dir               # recursive download / upload
mget "*.log"       ;  mput "*.jpg"             # glob download / upload
rename old new     ;  rm file                  # remote file management
!ls -la                                        # run a local command
sftp -b commands.txt user@host                 # batch/non-interactive
```

For related material, see [Chroot SFTP Setup](articles/chroot-sftp-setup.md) and the [rclone Cheatsheet](articles/rclone-cheatsheet.md).
