# rsync Cheatsheet

`rsync` copies and synchronizes files locally or between hosts, transferring **only the differences** between source and destination. That delta algorithm makes it far more efficient than a full copy for repeat syncs, and its options cover mirroring, backups, resumable transfers, and bandwidth limits. It usually runs over SSH for remote transfers.

For related tools, see the [rclone Cheatsheet](articles/rclone-cheatsheet.md) (cloud/object storage), the [SFTP Cheatsheet](articles/sftp-cheatsheet.md) (interactive transfer), and the [tar Cheatsheet](articles/tar-cheatsheet.md) (archiving).

## Syntax

```text
rsync [options] SOURCE... DESTINATION
```

Source and destination can be local paths or remote `user@host:path`. `rsync` can't sync two remote hosts in one command (source **or** dest may be remote, not both, in older versions).

## The Trailing Slash Rule

The single most important `rsync` gotcha: a trailing `/` on the **source** means "the contents of this directory", while no slash means "this directory itself".

```sh
rsync -a src/  dest/     # copies the CONTENTS of src into dest/
rsync -a src   dest/     # creates dest/src/ (copies the directory itself)
```

Get this wrong and you either nest a directory one level too deep or dump files where you meant to place a folder. The destination's trailing slash doesn't matter.

## Everyday Usage

```sh
rsync -av src/ dest/                         # archive + verbose, local copy
rsync -av src/ user@host:/dest/              # push to a remote host (over SSH)
rsync -av user@host:/src/ dest/              # pull from a remote host
rsync -avz src/ user@host:/dest/             # add compression for the transfer
rsync -av --progress src/ dest/              # show per-file progress
```

`-a` (archive) is the workhorse: it implies `-rlptgoD` — recurse, and preserve symlinks, permissions, timestamps, group, owner, and device/special files. It's what you want for a faithful copy in almost all cases.

## Core Options

| Option | Meaning |
|--------|---------|
| `-a` | Archive mode: recursive + preserve most attributes (`-rlptgoD`) |
| `-r` | Recurse into directories |
| `-v` / `-vv` | Verbose (more v's = more detail) |
| `-z` | Compress data during transfer |
| `-P` | `--partial --progress`: show progress and keep partial files (resumable) |
| `-h` | Human-readable sizes |
| `-n` / `--dry-run` | Show what would happen without doing it |
| `--delete` | Delete files in dest that no longer exist in source (mirror) |
| `--exclude PATTERN` | Skip files matching PATTERN |
| `--include PATTERN` | Force-include (used with `--exclude`) |
| `-e "ssh ..."` | Set the remote shell/command (e.g. custom SSH port) |
| `--bwlimit=RATE` | Cap bandwidth (e.g. `--bwlimit=10m`) |
| `-u` / `--update` | Skip files that are newer on the destination |
| `-c` / `--checksum` | Compare by checksum, not size+mtime |
| `--partial` | Keep partially transferred files (resume later) |
| `-x` / `--one-file-system` | Don't cross filesystem boundaries |

## Mirroring with --delete

To make the destination an exact copy of the source (removing extra files), add `--delete`. **Always dry-run first** — `--delete` can remove a lot if the source path is wrong:

```sh
rsync -av --delete src/ dest/ --dry-run      # preview deletions and changes
rsync -av --delete src/ dest/                # then run for real
```

> A trailing-slash mistake combined with `--delete` is the classic way to wipe a destination. Dry-run every mirror until the trailing slashes and paths are confirmed.

## Excluding Files

```sh
rsync -av --exclude '*.log' src/ dest/                 # skip one pattern
rsync -av --exclude '*.log' --exclude 'cache/' src/ dest/
rsync -av --exclude-from=excludes.txt src/ dest/       # patterns from a file

# include one thing but exclude the rest
rsync -av --include '*.conf' --exclude '*' src/ dest/
```

Patterns are relative to the source root; a trailing `/` matches directories. Order matters: the first matching `--include`/`--exclude` wins.

## Over SSH (Ports, Keys, Progress)

`rsync` uses SSH by default for remote transfers. Customize the SSH command with `-e`:

```sh
rsync -avz -e "ssh -p 2222" src/ user@host:/dest/      # non-standard SSH port
rsync -avz -e "ssh -i ~/.ssh/backup_key" src/ user@host:/dest/
rsync -avzP src/ user@host:/dest/                       # progress + resumable
```

## Backups and Resuming

```sh
# resumable large transfer — keeps partial files, shows progress
rsync -avzP bigfile user@host:/dest/

# timestamped backup of files being replaced/deleted
rsync -av --backup --backup-dir=/backups/$(date +%F) src/ dest/

# limit bandwidth to 5 MB/s so a sync doesn't saturate the link
rsync -avz --bwlimit=5m src/ user@host:/dest/
```

`-P` makes transfers resumable: if interrupted, rerunning the same command continues from the partial file rather than restarting.

## Dry Run — Always Preview

```sh
rsync -avn --delete src/ dest/               # -n = dry run; shows planned changes
rsync -avni src/ dest/                       # -i itemizes what would change per file
```

The `-i` (itemize) output uses codes like `>f+++++++++` (new file being sent) and `>f.st......` (file differs in size/time) — invaluable for understanding exactly what a sync will do before committing.

## Notes and Gotchas

- **Trailing slash on the source** decides contents-of vs the-directory-itself. This is the #1 rsync mistake.
- **`--delete` mirrors** — extra files in the destination are removed. Dry-run first, every time.
- **`-a` doesn't include `-z` or `-H`.** Add `-z` for compression and `-H` to preserve hard links if you need them.
- **`-c` (checksum) is slow** — it reads every file. The default size+mtime comparison is fast and correct for most cases; use `-c` only when timestamps are unreliable.
- **rsync must exist on both ends** for remote transfers (it runs a remote `rsync` over SSH).
- **Preserving ownership** (`-o`/`-g`, part of `-a`) needs root on the destination; without it, files land owned by the transfer user.

## Quick Reference

```sh
rsync -av src/ dest/                         # local copy (archive)
rsync -avz src/ user@host:/dest/             # push over SSH, compressed
rsync -avz user@host:/src/ dest/             # pull over SSH
rsync -av --delete src/ dest/                # mirror (removes extras)
rsync -avn --delete src/ dest/               # DRY RUN a mirror first
rsync -av --exclude '*.log' src/ dest/       # exclude a pattern
rsync -avzP -e "ssh -p 2222" big user@host:/d/   # port + resume + progress
rsync -avz --bwlimit=5m src/ user@host:/dest/    # throttle bandwidth
```

For related material, see the [rclone Cheatsheet](articles/rclone-cheatsheet.md), the [SFTP Cheatsheet](articles/sftp-cheatsheet.md), and the [tar Cheatsheet](articles/tar-cheatsheet.md).
