# tar Cheatsheet

`tar` (tape archive) bundles many files into a single archive, optionally compressed. The three operations you use daily are **create** (`c`), **extract** (`x`), and **list** (`t`), combined with a compression flag (`z` gzip, `j` bzip2, `J` xz) and `f` to name the file. This cheatsheet covers those plus the less-obvious bits: stripping path components, appending, diffing, and streaming backups over SSH.

For related file tooling, see the [find Cheatsheet](articles/find-cheatsheet.md), the [xargs Cheatsheet](articles/xargs-cheatsheet.md), and the [SFTP Cheatsheet](articles/sftp-cheatsheet.md).

## The Flag Grammar

Most `tar` commands are one operation letter plus modifiers. The classic bundling (`czvf`) reads as: create, gzip, verbose, file.

| Flag | Meaning |
|------|---------|
| `c` | Create an archive |
| `x` | Extract |
| `t` | List (table of contents) |
| `r` | Append files (uncompressed archives only) |
| `d` | Diff archive against the filesystem |
| `f FILE` | Use archive FILE (`-` = stdin/stdout) |
| `v` | Verbose — list files as processed |
| `z` | gzip compression (`.gz`) |
| `j` | bzip2 compression (`.bz2`) |
| `J` | xz compression (`.xz`) |
| `-C DIR` | Change to DIR before acting |
| `-P` | Keep/use absolute paths (don't strip leading `/`) |
| `--strip-components=N` | Drop N leading path components on extract |

> `f` must come **last** among the bundled letters, because the next argument is the filename: `czvf home.tar.gz` works, `cfzv home.tar.gz` does not. Leading dashes are optional in the classic style (`tar czvf` = `tar -czvf`).

## Create

```sh
tar cvf home.tar /home                     # uncompressed
tar czvf home.tar.gz /home                 # gzip
tar cjvf home.tar.bz2 /home                # bzip2
tar cJvf home.tar.xz /home                 # xz (smaller, slower)

# archive just one subdirectory of a tree (cd there first with -C)
tar -C /home -czvf home-daniel.tar.gz daniel

# timestamped archive name
tar czvf home-$(date +%Y%m%d).tar.gz /home
```

> By default `tar` **strips the leading `/`** and stores paths as relative (`home/daniel/...`), printing "Removing leading `/` from member names". That's what makes archives safe to extract anywhere. Use `-P` only if you deliberately want absolute paths baked in (see below).

## Extract

```sh
tar xvf home.tar                           # uncompressed, into current dir
tar xzvf home.tar.gz                       # gzip
tar xjvf home.tar.bz2                      # bzip2
```

Modern GNU `tar` auto-detects compression on extract, so `tar xvf home.tar.gz` also works — but naming the flag (`z`/`j`/`J`) is explicit and portable.

### Extract to a specific location

Because paths are stored relative, extract from the intended root. Two equivalent ways:

```sh
cd / && tar xzvf home.tar.gz               # cd first
tar xzvf home.tar.gz -C /                  # or let tar change dir
```

### Extract only part of an archive

```sh
# a single directory, dropping its leading path component
tar xzvf home.tar.gz home/daniel --strip-components=1

# one file, keeping its directory structure
tar xzvf home.tar.gz home/dcorneschi/.ssh/authorized_keys

# one file, flattened into the current dir (drop 3 leading components)
tar xzvf home.tar.gz home/dcorneschi/.ssh/authorized_keys --strip-components=3
```

`--strip-components=N` removes N leading directory levels from each extracted path — handy for pulling a nested file into the current directory, or unpacking a `project-1.2.3/` tarball straight into `.`.

## List Contents

```sh
tar tvf home.tar                           # uncompressed
tar tzvf home.tar.gz                       # gzip
tar tjvf home.tar.bz2                      # bzip2
```

`t` without `v` prints just names; with `v` it adds permissions, owner, size, and date — a quick way to inspect an archive before extracting.

## Append and Update

`r` appends to an **uncompressed** archive. You cannot append to a gzip/bzip2/xz archive directly (the compression wraps the whole stream):

```sh
tar rvf home.tar /tmp /etc/fstab           # add to an uncompressed .tar
```

Workaround for a compressed archive — decompress, append, recompress:

```sh
gzip -d home.tar.gz                        # -> home.tar
tar rvf home.tar /etc
gzip home.tar                              # -> home.tar.gz
```

## Inspect and Verify

```sh
# estimate the compressed size without writing a file
tar czf - /home | wc -c                    # bytes of the gzip stream

# diff an archive against the current filesystem
tar dzf home.tar.gz -C /                   # reports files that changed
```

`d` (diff/compare) lists members whose size, mode, or mtime differ from what's on disk — useful to see what changed since a backup.

## Absolute Paths (-P)

`-P` disables the default stripping of the leading `/`, storing (and restoring) absolute paths:

```sh
tar czPvf home.tar.gz /home                # store absolute /home/... paths
tar xzPvf home.tar.gz                      # restore to those absolute paths
```

> Use `-P` with care: an archive with absolute paths extracts **over the real system paths** regardless of your current directory, which can overwrite live files. The relative-path default (extract with `-C /`) is safer and more flexible. Only reach for `-P` when you specifically need the paths pinned.

## Backups Over SSH

Stream an archive to another host without a temp file by writing to stdout (`f -`) and piping through `ssh`:

```sh
# push a backup to a remote server
tar czvf - /home | ssh root@192.168.1.22 "cat > /root/$(hostname)-home.tar.gz"

# restore it back, extracting at the root
ssh root@192.168.1.22 "cat /root/server_name-home.tar.gz" | tar xzvf - -C /
```

The `-` as the filename makes `tar` read from / write to the pipe. See the [SFTP Cheatsheet](articles/sftp-cheatsheet.md) for a file-transfer alternative.

## Key Takeaways

- Bundle the operation + compression + `f`: `czvf` = create/gzip/verbose/file; `f` comes last.
- Compression: `z` gzip, `j` bzip2, `J` xz. GNU `tar` auto-detects on extract.
- `tar` stores **relative** paths (strips leading `/`) — extract with `-C /` to restore in place; `-P` keeps absolute paths but is riskier.
- `--strip-components=N` drops leading path levels on extract; name members to extract only part of an archive.
- You can't append (`r`) to a compressed archive — decompress, append, recompress.
- Stream over SSH with `f -` and a pipe for tempfile-free remote backups.

## Quick Reference

```sh
tar czvf a.tar.gz DIR                      # create gzip archive
tar xzvf a.tar.gz -C /dest                 # extract to /dest
tar tzvf a.tar.gz                          # list contents
tar -C /home -czvf u.tar.gz user           # archive one subdir
tar xzvf a.tar.gz path/to/file --strip-components=2   # extract one file, flattened
tar czf - DIR | wc -c                      # estimate compressed size
tar dzf a.tar.gz -C /                      # diff against filesystem
tar czvf - DIR | ssh host "cat > bak.tar.gz"          # remote backup
```

For related material, see the [find Cheatsheet](articles/find-cheatsheet.md), the [xargs Cheatsheet](articles/xargs-cheatsheet.md), and the [SFTP Cheatsheet](articles/sftp-cheatsheet.md).
