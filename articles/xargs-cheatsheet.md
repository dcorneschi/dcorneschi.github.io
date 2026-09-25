# xargs Cheatsheet

`xargs` reads items from **standard input** and builds command lines from them, running a command with those items as arguments. It's the glue between commands that *produce* lists (like `find`, `ls`, `grep -l`) and commands that *take* arguments (like `rm`, `cp`, `grep`). It also batches efficiently — instead of one process per item, `xargs` packs many items into as few invocations as the argument limit allows.

For related tools, see the [find Cheatsheet](articles/find-cheatsheet.md), the [tr Cheatsheet](articles/tr-cheatsheet.md), and the [sed Cheatsheet](articles/sed-cheatsheet.md).

## Syntax

```text
command | xargs [options] [command [initial-args]]
```

Without a command, `xargs` defaults to `echo`. It appends the input items **after** any initial arguments you give.

## Core Options

| Option | Meaning |
|--------|---------|
| `-0` | Input items are NUL-separated (pair with `find -print0`) |
| `-d DELIM` | Use a custom delimiter instead of whitespace/newline |
| `-n N` | At most N arguments per command invocation |
| `-I {}` | Replace `{}` with each input item (implies one item per run) |
| `-P N` | Run up to N invocations in parallel |
| `-t` | Print each command before running it |
| `-p` | Prompt for confirmation before each command |
| `-r` | Don't run the command at all if input is empty (GNU) |
| `-s SIZE` | Limit the command-line length in bytes |
| `-a FILE` | Read items from FILE instead of stdin |

## The Two Safety Rules

Two flags prevent the most common `xargs` bugs. Use them by reflex:

```sh
# 1. NUL-separate to survive spaces/newlines in filenames
find . -name '*.tmp' -print0 | xargs -0 rm

# 2. Don't run on empty input (GNU xargs)
find . -name '*.nomatch' | xargs -r rm
```

Without `-0`, a filename containing a space or newline is split into multiple arguments — which, with `rm`, can delete the wrong files. Without `-r`, some `xargs` implementations run the command once with no arguments when input is empty (e.g. `rm` with no args errors; worse commands may do something unwanted).

## Controlling How Arguments Are Passed

### Default: pack many per invocation

```sh
echo "a b c" | xargs rm          # runs: rm a b c   (one process)
```

### One (or N) at a time (-n)

```sh
echo "a b c" | xargs -n1 rm      # rm a  /  rm b  /  rm c
seq 1 10   | xargs -n3 echo      # 1 2 3 / 4 5 6 / 7 8 9 / 10
```

### Placeholder substitution (-I)

`-I {}` puts each item at a specific spot in the command (not just at the end), and runs once per item:

```sh
ls *.txt | xargs -I {} cp {} backup/
cat urls.txt | xargs -I {} curl -s {}
```

> `-I` processes **one item per line of input** and implies `-n1`, so it's inherently slower than batching. Use it when the item must appear mid-command; use plain batching (or `-n`) when arguments just go at the end.

### Custom delimiter (-d)

```sh
echo "apple,banana,cherry" | xargs -d ',' -n1 echo
```

### Feeding one-per-line input

Commands like `find -print0` are NUL-safe, but for whitespace-joined lists convert spaces to newlines first (see the [tr Cheatsheet](articles/tr-cheatsheet.md)):

```sh
echo "item1 item2 item3" | tr ' ' '\n' | xargs -I {} echo "Processing: {}"
```

## Parallel Execution (-P)

`-P N` runs up to N invocations concurrently — a simple way to parallelize independent work:

```sh
# resize images, 4 at a time
find . -name '*.jpg' -print0 | xargs -0 -P4 -I {} convert {} -resize 800x600 out/{}

# download in parallel, one URL per process
cat urls.txt | xargs -n1 -P5 wget

# compress files 8 at a time
ls *.txt | xargs -P8 -n1 gzip
```

> Parallel output can interleave. `-P` is best for independent tasks; combine with `-n1` (or `-I`) so each process gets a discrete unit of work.

## Preview and Confirm

Before destructive runs, see or approve the commands:

```sh
find . -name '*.log' | xargs -t gzip     # print each command, then run it
find . -name '*.tmp' | xargs -p rm       # prompt y/n before running
```

## Common Uses

### File operations

```sh
find . -name '*.sh'  -print0 | xargs -0 chmod +x           # make scripts executable
find . -name '*.log' -print0 | xargs -0 -I {} mv {} archive/
find . -type d -empty -print0 | xargs -0 rmdir             # remove empty dirs
ls -d */ | xargs du -sh                                    # size of each subdir
```

### Search across files (grep)

```sh
find . -name '*.py' -print0 | xargs -0 grep -l 'import pandas'   # files that match
find . -name '*.py' -print0 | xargs -0 grep -Hn 'TODO'          # path:line for each hit
```

### Bulk edits (sed)

```sh
find . -name '*.txt' -print0 | xargs -0 sed -i 's/old/new/g'
```

### Running a shell per item

When you need pipes or multiple commands per item, wrap in `sh -c`:

```sh
ls *.txt | xargs -I {} sh -c 'echo "== {} =="; wc -l < "{}"'
```

## find: xargs vs -exec

Both apply a command to found files. Rough guidance:

```sh
find . -name '*.tmp' -exec rm {} \;      # one rm per file (slow)
find . -name '*.tmp' -exec rm {} +       # batched by find itself (fast, no pipe)
find . -name '*.tmp' -print0 | xargs -0 rm   # batched via xargs (fast; adds -P option)
```

`-exec … +` and `xargs` both batch; `-exec` needs no pipe and no `-0`, while `xargs` adds parallelism (`-P`) and finer batch control (`-n`, `-s`). See the [find Cheatsheet](articles/find-cheatsheet.md).

## Examples by Domain

### System administration

```sh
ps aux | grep '[p]ython' | awk '{print $2}' | xargs -r kill     # kill by name
find /var/log -name '*.log' -print0 | xargs -0 chown root:root
echo "nginx mysql" | xargs -n1 systemctl restart               # restart services
```

> Prefer `pgrep`/`pkill` over the `ps | grep | awk | xargs kill` chain when available — it's less fragile. The `[p]ython` trick stops grep from matching its own process.

### Git

```sh
ls -d */ | xargs -I {} git -C {} pull                          # pull each repo
git branch --merged | grep -vE 'main|master' | xargs -r git branch -d
```

### Kubernetes

```sh
kubectl get nodes -o name | xargs -I {} kubectl get pods -A --field-selector spec.nodeName={}
kubectl get nodes -l env=staging -o name | xargs -I {} kubectl drain {} --ignore-daemonsets
kubectl get deploy -n prod -o name | xargs -I {} kubectl rollout restart {} -n prod
```

### AWS CLI

```sh
aws ec2 describe-instances --filters "Name=tag:Environment,Values=test" \
  --query 'Reservations[].Instances[].InstanceId' --output text \
  | xargs -n1 aws ec2 terminate-instances --instance-ids
```

## When a while-read Loop Is Clearer

`xargs -I {}` is concise but limited to substituting one placeholder. For multi-step logic per item, a loop is often more readable and safer with odd filenames:

```sh
find . -name '*.log' -print0 | while IFS= read -r -d '' f; do
    echo "processing: $f"
    gzip "$f"
done
```

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Filenames with spaces/newlines split apart | `find -print0` + `xargs -0` |
| Command runs on empty input | `xargs -r` (GNU) |
| "Argument list too long" | `xargs` already chunks; tune with `-n` or `-s` |
| `-I {}` unexpectedly slow | It forces one item per run; drop it to batch |
| Parallel output interleaves | Expected with `-P`; give each process one unit via `-n1`/`-I` |

## Key Takeaways

- `xargs` turns stdin into command arguments and batches them into few invocations.
- **Always** use `find -print0 | xargs -0` for filenames, and `-r` to skip empty input.
- `-n N` sets args per run, `-I {}` places each item mid-command (one per run), `-P N` parallelizes.
- Preview with `-t` or confirm with `-p` before destructive operations.
- For per-item pipelines or multi-step logic, a `while IFS= read -r -d ''` loop is clearer than `-I {}`.

## Quick Reference

```sh
find . -name '*.tmp' -print0 | xargs -0 -r rm     # safe delete
echo "a b c" | xargs -n1 echo                     # one arg per run
ls *.txt | xargs -I {} cp {} backup/              # placeholder substitution
cat urls.txt | xargs -n1 -P5 wget                 # parallel, one per process
find . -name '*.py' -print0 | xargs -0 grep -l x  # search across files
seq 1 10 | xargs -n3 echo                          # 3 args per line
echo "a,b,c" | xargs -d, -n1 echo                  # custom delimiter
find . -name '*.log' | xargs -t gzip               # preview commands
```

For related material, see the [find Cheatsheet](articles/find-cheatsheet.md), the [tr Cheatsheet](articles/tr-cheatsheet.md), and the [sed Cheatsheet](articles/sed-cheatsheet.md).
