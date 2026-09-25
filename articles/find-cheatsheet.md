# find Cheatsheet

`find` walks a directory tree and matches files by name, type, size, time, ownership, and permissions, then optionally acts on the matches (`-exec`, `-delete`, pipe to `xargs`). This cheatsheet collects the most useful patterns for day-to-day system administration.

> When scanning from `/`, pseudo-filesystems like `/proc` produce noise and permission errors. Append `2>/dev/null` to discard stderr, e.g. `find / -perm /6000 -type f 2>/dev/null`.

For related text-processing tools, see the [awk Cheatsheet](articles/awk-cheatsheet.md), [sed Cheatsheet](articles/sed-cheatsheet.md), and [Linux Processes and Signals Cheatsheet](articles/ps-cheatsheet.md).

## Find by Name

```sh
find / -name fstab              # exact name, case-sensitive
find / -iname fstab             # case-insensitive
find / -name '*.rpmsave'        # by extension (quote the glob!)
```

> Always **quote** wildcard patterns (`'*.rpmsave'`). Unquoted, the shell may expand `*` before `find` ever sees it, causing wrong or failing matches.

## Find by Type

```sh
find / -type f                  # regular files
find / -type d                  # directories
find / -type l                  # symbolic links
```

Common type letters: `f` file, `d` directory, `l` symlink, `b` block device, `c` char device, `p` named pipe (FIFO), `s` socket.

```sh
find / -type f -executable      # regular files that are executable
```

### Negating a test (! / -not)

Prefix any test with `!` (or `-not`) to invert it:

```sh
find /path ! -name '*.tmp'      # everything except *.tmp
find /path -type f ! -perm 644  # files whose mode is not exactly 644
```

## Limiting Scope: -maxdepth, -mindepth, -prune

Narrowing the search speeds things up and avoids irrelevant matches.

```sh
find /path -maxdepth 2 -name '*.txt'   # search at most 2 levels deep
find /path -mindepth 2 -name '*.txt'   # skip the top level, start at level 2
```

`-prune` skips entire directories rather than descending into them — the classic use is ignoring `.git` or `node_modules`:

```sh
# find *.py but never descend into any .git directory
find /path -name .git -prune -o -name '*.py' -print
```

> The `-prune` idiom reads as: "if the entry is `.git`, prune it; **otherwise** (`-o`) if it matches `*.py`, print it." The trailing `-print` is required — once you use `-o`, find no longer prints by default.

## Empty Files and Directories

```sh
find / -depth -type f -empty                     # empty files
find / -depth -type d -empty                     # empty directories
find . -type d -empty -exec rmdir -v {} +        # delete empty directories
```

`-depth` processes a directory's contents before the directory itself, which is what you want when removing empty trees.

## Find by Time

`find` compares timestamps in **days** (`-mtime`, `-atime`, `-ctime`) or **minutes** (`-mmin`, `-amin`, `-cmin`). A leading `+` means "more than", `-` means "less than", and no sign means "exactly".

```sh
find /etc -mtime 0              # modified in the last 24 hours
find /home -atime 0            # accessed in the last 24 hours
find /etc -mmin 1              # modified exactly 1 minute ago
find /etc -mmin -30           # modified less than 30 minutes ago
find /etc -mmin +5 -mmin -10  # modified between 5 and 10 minutes ago
```

| Test | Unit | Meaning |
|------|------|---------|
| `-mtime N` | days | modification time |
| `-atime N` | days | access time |
| `-ctime N` | days | inode change time |
| `-mmin N` | minutes | modification time |
| `-amin N` | minutes | access time |
| `-cmin N` | minutes | inode change time |

### Relative to a specific date (-newermt)

`-newermt` compares against an explicit timestamp; negate with `-not` (or `!`) for "older than":

```sh
# delete files NOT modified after 2025-01-26 (i.e. older than that date)
find /mnt/pve/datastore/dump -type f -not -newermt "2025-01-26" -delete
```

The `-newerXY` family lets you pick which timestamps to compare. `X` is the file's timestamp, `Y` the reference:

| Letter | Reference timestamp |
|--------|---------------------|
| `a` | access time |
| `B` | birth (creation) time |
| `c` | inode status-change time |
| `m` | modification time |
| `t` | `Y` only — interpret the reference as a literal time string |

So `-newermt` = compare **m**odification time against a literal **t**ime.

## Find by Owner and Group

```sh
find / -user daniel                       # owned by user daniel
find / -user daniel -group daniel         # owned by user AND group daniel
find / -user daniel -o -user adina        # owned by daniel OR adina
find / -uid 101                           # by numeric user ID
find / -gid 101                           # by numeric group ID
find / -nouser -o -nogroup                # unowned (orphaned UID/GID)
```

`-o` is the OR operator; criteria are otherwise AND'd together.

## Find by Size

Size suffixes: `c` bytes, `k` KiB, `M` MiB, `G` GiB. `+` means larger than, `-` smaller than, none means exactly.

```sh
find / -size 868c              # exactly 868 bytes
find /etc -size -1024k         # smaller than 1024 KiB
find /etc -size +1M            # larger than 1 MiB
```

## Find by Permissions

Three ways to match the mode:

- `-perm MODE` — exact match
- `-perm -MODE` — all of these bits set (AND)
- `-perm /MODE` — any of these bits set (OR)

```sh
find / -type f -perm 755            # files with exactly 755
find / -type d -perm 755            # directories with exactly 755
find /etc -type f -perm /002        # files world-writable ("other" write bit)
find /etc -type d -perm /002        # directories world-writable
```

### SUID / SGID binaries

The old `-perm +MODE` syntax is deprecated; use `-perm /MODE` instead.

```sh
find / -perm /4000 -type f 2>/dev/null                    # SUID
find / -perm /2000 -type f 2>/dev/null                    # SGID
find / -perm /6000 -type f -exec ls -ld {} \; 2>/dev/null # SUID or SGID
```

Auditing SUID/SGID binaries is a common security task — an unexpected entry can be a privilege-escalation vector.

## Acting on Matches

### -exec

`{}` is replaced by each match. Terminate with `\;` (runs once per file) or `+` (batches many files into one invocation, faster):

```sh
find . -type f -exec chmod 664 {} \;      # one chmod per file
find . -type f -exec chmod 664 {} +       # batched — far fewer processes
```

### -ok (confirm each action)

`-ok` is like `-exec` but prompts for confirmation before running the command on each match — a safety net for destructive operations:

```sh
find /path -name '*.tmp' -ok rm {} \;     # ask y/n before removing each file
```

### -printf (custom output)

`-printf` prints selected fields per match. The most useful pattern is sorting by timestamp — `%T@` is the modification time as a Unix epoch, `%p` the path:

```sh
find /path -type f -printf '%T@ %p\n' | sort -nr | head -5   # 5 newest files
find /path -type f -printf '%T@ %p\n' | sort -n  | head -5   # 5 oldest files
find /path -type f -printf '%s %p\n'  | sort -nr | head      # largest by bytes
```

Common `-printf` directives: `%p` path, `%f` filename, `%s` size in bytes, `%T@` mtime (epoch), `%u` owner, `%g` group, `%m` octal permissions.

### Multiple actions in one pass

Chain several `-exec` clauses; they run left to right per match:

```sh
find . -type f \( -name 'daniel*' -o -name 'adina*' -o -name 'pisu*' \) \
  -exec ls -l {} \; -exec chown root {} \; -exec chmod 700 {} \;
```

### Matching several name patterns

Group alternatives with escaped parentheses `\( ... \)`:

```sh
find . -type f \( -name daniel -o -name adina -o -name pisu \) -exec ls -l {} \;
find . -type f \( -name 'daniel*' -o -name 'adina*' -o -name 'pisu*' \) -exec gzip {} \;
```

### -delete

```sh
find . -type f \( -name 'daniel*' -o -name 'adina*' -o -name 'pisu*' \) -delete
```

> `-delete` implies `-depth`. Test first by replacing `-delete` with `-print` to confirm the match set before you remove anything.

## Setting Permissions by Type

The clean approach — recurse and act by type:

```sh
find . -type f -exec chmod 664 {} \;      # files only
find . -type d -exec chmod 775 {} \;      # directories only
```

### Alternatives with xargs

For large trees, `-print0 | xargs -0` is efficient and NUL-safe (handles spaces/newlines in names):

```sh
find /path/to/base/dir -type d -print0 | xargs -0 chmod 755
find /path/to/base/dir -type f -print0 | xargs -0 chmod 644
```

Avoid the command-substitution form `chmod 755 $(find ... -type d)` — it breaks on filenames with spaces and can blow past the argument-length limit on big trees. Prefer `-exec ... +` or `xargs -0`.

## Combining find with Other Commands

### Find files containing text (grep)

```sh
find /path -type f -exec grep -l 'search_term' {} +   # list matching files
find /path -name '*.py' -exec grep -in 'TODO' {} +    # case-insensitive, line numbers
find /src -type f -exec grep -Hn 'TODO\|FIXME' {} +   # path + line for each hit
```

`grep -l` prints only the filenames; `-H` forces the filename prefix; `-n` adds line numbers.

### Process each file in a loop (while read)

Use NUL-separated input so filenames with spaces or newlines survive:

```sh
find /path -name '*.log' -print0 | while IFS= read -r -d '' file; do
    echo "Processing: $file"
    # commands using "$file"
done
```

### Bulk in-place edits (sed)

```sh
find /path -name '*.txt' -exec sed -i 's/old/new/g' {} +   # replace across files
find /src  -name '*.py'  -exec sed -i 's/2023/2024/g' {} +  # e.g. bump a year
```

### Archive matched files (tar)

Feed the file list to `tar` with `-T -` (read names from stdin):

```sh
find /path -type f -mtime -7 -print0 | tar --null -czf backup.tar.gz -T -
find /var/log -name '*.log' -mtime +30 -print0 | tar --null -czf old_logs.tar.gz -T -
```

### Find disk hogs

```sh
find / -type f -size +1G -exec ls -lh {} + 2>/dev/null       # files over 1 GiB
find /path -type f -printf '%s %p\n' | sort -nr | head -20   # 20 largest by bytes
```

### Count files by extension

```sh
find /path -type f | sed 's/.*\.//' | sort | uniq -c | sort -nr
```

### Total size of matched files

```sh
find /path -type f -name '*.log' -exec du -ch {} + | tail -1   # grand total
```

## Flags Reference

| Flag | Description |
|------|-------------|
| `-name` / `-iname` | Match filename (case-sensitive / insensitive) |
| `!` / `-not` | Negate the following test |
| `-o` | OR two tests (default between tests is AND) |
| `-type` | File type: `f` file, `d` dir, `l` symlink, `b`/`c` device, `p` FIFO, `s` socket |
| `-executable` | File is executable |
| `-empty` | File or directory is empty |
| `-size` | Size: `c` bytes, `k`, `M`, `G`; `+`/`-` for more/less |
| `-mtime` / `-atime` / `-ctime` | Modify / access / change time in days |
| `-mmin` / `-amin` / `-cmin` | Same, in minutes |
| `-newermt` | Compare mtime against a literal date |
| `-perm` | Permissions: `MODE` exact, `-MODE` all bits, `/MODE` any bit |
| `-user` / `-group` | Owner / group by name |
| `-uid` / `-gid` | Owner / group by numeric ID |
| `-nouser` / `-nogroup` | No matching passwd/group entry |
| `-maxdepth` / `-mindepth` | Limit recursion depth |
| `-prune` | Don't descend into matched directories |
| `-exec ... \;` | Run command once per match |
| `-exec ... +` | Run command with matches batched |
| `-ok` | Like `-exec` but prompt before each |
| `-delete` | Delete matches (implies `-depth`) |
| `-print` / `-print0` | Print path (newline / NUL separated) |
| `-printf` | Print custom fields (`%p`, `%f`, `%s`, `%T@`, …) |

## Key Takeaways

- Quote glob patterns (`-name '*.log'`) so the shell doesn't expand them first.
- Time tests: `+N` = more than, `-N` = less than, `N` = exactly; days for `-mtime`, minutes for `-mmin`.
- Permissions: `-perm MODE` exact, `-perm -MODE` all bits, `-perm /MODE` any bit (use `/` instead of deprecated `+`).
- `-exec {} +` and `xargs -0` batch work efficiently; `-exec {} \;` runs once per file.
- Preview `-delete` with `-print` first; append `2>/dev/null` to silence `/proc` and permission noise.
- Narrow the search with `-maxdepth`/`-mindepth`, and skip whole subtrees with `-prune` (e.g. `.git`).
- Use `-printf '%T@ %p\n' | sort` to rank matches by time, and `-ok` to confirm destructive actions per file.

## Quick Reference

```sh
find / -iname 'name'                       # by name, case-insensitive
find / -type f -empty                      # empty files
find /etc -mmin -30                        # modified in last 30 min
find / -user daniel -group daniel          # by owner and group
find /etc -size +1M                        # larger than 1 MiB
find / -type f -perm /002 2>/dev/null      # world-writable files
find / -perm /6000 -type f 2>/dev/null     # SUID/SGID binaries
find . -type f -exec chmod 664 {} +        # chmod all files (batched)
find DIR -type f -not -newermt "2025-01-26" -delete   # older than a date
find /path -maxdepth 2 -name '*.txt'       # limit recursion depth
find /path -name .git -prune -o -name '*.py' -print   # skip .git subtrees
find /path -type f -exec grep -l 'text' {} +          # files containing text
find /path -type f -printf '%T@ %p\n' | sort -nr | head   # newest files
```

For related material, see the [awk Cheatsheet](articles/awk-cheatsheet.md), the [sed Cheatsheet](articles/sed-cheatsheet.md), and the [Linux Processes and Signals Cheatsheet](articles/ps-cheatsheet.md).
