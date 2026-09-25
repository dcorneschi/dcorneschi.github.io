# Deleting Files by Age or Date

A recurring housekeeping task: remove files older (or newer) than some age, or before/after a specific date — log rotation, cache cleanup, purging temp files. `find` is the right tool for it, but there are pure-shell alternatives when `find` isn't available or you want a quick one-off. This guide collects the reliable patterns and, just as important, the safe way to preview before you delete.

For the general command reference, see the [find Cheatsheet](articles/find-cheatsheet.md); for related tooling, the [sed Cheatsheet](articles/sed-cheatsheet.md) and [awk Cheatsheet](articles/awk-cheatsheet.md).

> **Deletion is irreversible.** Always run the preview form first (drop `-delete`/`-exec rm` and just list matches) and confirm the set is what you expect. See [Safe Preview First](#safe-preview-first).

## The find Approach (Recommended)

`find` selects by age or timestamp and acts in one pass. Two ways to delete:

- `-delete` — built-in, no subprocess, safe with odd filenames.
- `-exec rm {} +` — spawns `rm` in batches; use when you need `rm` flags like `-v` or `-i`.

Prefer `-delete` unless you specifically need `rm`'s behavior.

### By age in days (-mtime / -atime)

`-mtime` counts 24-hour periods: `+N` = more than N days ago, `-N` = less than N days ago, `N` = exactly N.

```bash
find /path -type f -mtime +30 -delete    # modified more than 30 days ago
find /path -type f -mtime +7  -delete    # older than 7 days
find /path -type f -mtime -7  -delete    # modified within the last 7 days
find /path -type f -atime +30 -delete    # not accessed in over 30 days
```

### By specific date (-newermt)

`-newermt` compares modification time against a literal date; negate with `!` (or `-not`) for "before":

```bash
# before a date (older than 2024-01-01)
find /path -type f ! -newermt "2024-01-01" -delete

# after a date
find /path -type f -newermt "2024-01-01" -delete

# on a single day — after start-of-day AND before end-of-day
find /path -type f -newermt "2024-01-01 00:00:00" \
                 ! -newermt "2024-01-02 00:00:00" -delete
```

### By creation/birth time (-newerct, where supported)

Birth time isn't recorded by every filesystem/kernel; test before relying on it.

```bash
find /path -type f ! -newerct "2024-01-01" -delete   # created before
find /path -type f  -newerct "2024-01-01" -delete   # created after
```

### Restricting to specific file types

Combine a `-name` test (group alternatives with `\( … \)`):

```bash
find /path -type f -name '*.log' -mtime +7 -delete
find /path -type f -name '*.tmp' -mtime +1 -delete
find /path -type f \( -name '*.jpg' -o -name '*.png' -o -name '*.gif' \) \
  ! -newermt "2024-01-01" -delete
```

## Using -exec rm and Its Flags

When you want `rm`'s options, use `-exec`. Batch with `+` for efficiency:

```bash
find /path -type f -mtime +30 -exec rm {} \;      # one rm per file
find /path -type f -mtime +30 -exec rm -f {} +    # batched (far fewer processes)
find /path -type f -mtime +30 -exec rm -v {} \;   # verbose: print each removal
find /path -type f -mtime +30 -exec rm -iv {} \;  # prompt + verbose
```

| Form | Behavior |
|------|----------|
| `-exec rm {} \;` | Run `rm` once per file |
| `-exec rm {} +` | Batch many files into fewer `rm` calls (faster) |
| `-exec rm -f {} \;` | Force; ignore missing files, no prompts |
| `-exec rm -i {} \;` | Prompt before each deletion |
| `-exec rm -v {} \;` | Print each file as it's removed |
| `-delete` (built-in) | No `rm` subprocess; implies `-depth` |

## Using xargs for Very Large Sets

For huge match counts, pipe NUL-separated paths into `xargs`. **Always** use `-print0 | xargs -0` so names with spaces or newlines are handled correctly:

```bash
find /path -type f -mtime +30 -print0 | xargs -0 rm
find /path -type f -mtime +30 -print0 | xargs -0 rm -v
find /path -type f -mtime +30 -print0 | xargs -0 rm -i
```

## Safe Preview First

Before deleting, run the same selection **without** the delete action:

```bash
find /path -type f -mtime +30              # list matches
find /path -type f -mtime +30 -ls          # list with details (size, date)
find /path -type f -mtime +30 | wc -l      # count what would be deleted
```

Only once the list looks right, re-run with `-delete` or `-exec rm`. This one habit prevents the vast majority of accidental mass deletions.

## Pure-Shell Alternatives (No find)

When `find` isn't handy, a loop over `stat` timestamps works. These compare epoch seconds from `stat` against a cutoff from `date`. Quote `"$f"` always.

```bash
# older than 30 days (modification time)
for f in *; do
  [ -f "$f" ] && [ "$(stat -c %Y "$f")" -lt "$(date -d '30 days ago' +%s)" ] && rm -v "$f"
done

# newer than 7 days
for f in *; do
  [ -f "$f" ] && [ "$(stat -c %Y "$f")" -gt "$(date -d '7 days ago' +%s)" ] && rm "$f"
done

# before a specific date
for f in *; do
  [ -f "$f" ] && [ "$(stat -c %Y "$f")" -lt "$(date -d '2024-01-01' +%s)" ] && rm "$f"
done

# preview instead of deleting — swap rm for echo
for f in *; do
  [ -f "$f" ] && [ "$(stat -c %Y "$f")" -lt "$(date -d '30 days ago' +%s)" ] && echo "would delete: $f"
done
```

> The `for f in *` glob only covers the current directory's entries (not recursive) and skips dotfiles by default. For recursion or hidden files, `find` is the better tool. Avoid parsing `ls` output for this — filenames with spaces or newlines break `ls | awk` pipelines.

### Combined conditions

```bash
# in a date range (after 2024-01-01 AND before 2024-12-31)
for f in *; do
  [ -f "$f" ] || continue
  m=$(stat -c %Y "$f")
  [ "$m" -gt "$(date -d '2024-01-01' +%s)" ] && [ "$m" -lt "$(date -d '2024-12-31' +%s)" ] && rm "$f"
done

# size AND age (larger than 1 MiB and older than 30 days)
for f in *; do
  [ -f "$f" ] || continue
  [ "$(stat -c %s "$f")" -gt 1048576 ] && [ "$(stat -c %Y "$f")" -lt "$(date -d '30 days ago' +%s)" ] && rm "$f"
done
```

## Targeting a Specific Directory

```bash
find /path/to/dir -type f -mtime +30 -delete           # find takes the path directly
find . -type f -mtime +30 -delete                      # current directory

# for the shell-loop form, use a subshell so your cwd is unchanged
( cd /path/to/dir && for f in *; do
    [ -f "$f" ] && [ "$(stat -c %Y "$f")" -lt "$(date -d '30 days ago' +%s)" ] && rm "$f"
  done )
```

## macOS / BSD Differences

BSD `stat` and `date` differ from GNU (Linux):

| Task | Linux (GNU) | macOS (BSD) |
|------|-------------|-------------|
| File mtime (epoch) | `stat -c %Y` | `stat -f %m` |
| File atime (epoch) | `stat -c %X` | `stat -f %a` |
| File size (bytes) | `stat -c %s` | `stat -f %z` |
| Date N days ago | `date -d '30 days ago' +%s` | `date -v-30d +%s` |

```bash
# older than 30 days, macOS
for f in *; do
  [ -f "$f" ] && [ "$(stat -f %m "$f")" -lt "$(date -v-30d +%s)" ] && rm "$f"
done
```

`find -delete`, `-mtime`, and `-newermt` work the same on both, so the `find` forms above are the portable choice.

## Reference

### find time tests

| Test | Meaning |
|------|---------|
| `-mtime +N` | modified more than N days ago |
| `-mtime -N` | modified less than N days ago |
| `-atime +N` | accessed more than N days ago |
| `! -newermt "DATE"` | modified before DATE |
| `-newermt "DATE"` | modified after DATE |
| `! -newerct "DATE"` | created before DATE (if supported) |

### stat format specifiers (GNU)

| Spec | Meaning |
|------|---------|
| `%Y` | modification time (epoch seconds) |
| `%X` | access time (epoch seconds) |
| `%Z` | inode change time (epoch seconds) |
| `%s` | size in bytes |

## Key Takeaways

- Prefer `find … -delete` (or `-exec rm {} +`); it's clearer, recursive, and safe with awkward filenames.
- `-mtime +N`/`-N` = older/newer than N days; `-newermt "DATE"` and `! -newermt "DATE"` handle explicit dates.
- **Preview first** — run the selection without the delete action, and check `| wc -l`.
- With `xargs`, always use `-print0 | xargs -0`; don't parse `ls`.
- On macOS use `stat -f %m` and `date -v-30d`; the `find` forms are portable across Linux and BSD.

## Quick Reference

```bash
find /path -type f -mtime +30 -delete                 # older than 30 days
find /path -type f -mtime -7 -delete                  # newer than 7 days
find /path -type f ! -newermt "2024-01-01" -delete    # before a date
find /path -type f -name '*.log' -mtime +7 -delete    # by type + age
find /path -type f -mtime +30                         # PREVIEW (no delete)
find /path -type f -mtime +30 -print0 | xargs -0 rm   # very large sets
```

For related material, see the [find Cheatsheet](articles/find-cheatsheet.md), the [sed Cheatsheet](articles/sed-cheatsheet.md), and the [awk Cheatsheet](articles/awk-cheatsheet.md).
