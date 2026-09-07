# Bash Loops Guide: for, while, until, select

A practical tour of every looping construct in Bash — `for` (word lists, ranges, C-style, command output, arrays), `while` and `until`, the `select` menu, loop control with `break`/`continue` (including the `N` form for nested loops), and reading files into arrays with `mapfile`. For a deeper dive focused on `while read` patterns, see [Bash While Loop Examples](articles/bash-while-loops-examples.md).

> Bash has three looping keywords — `for`, `while`, and `until` — plus `select` for interactive menus. `for` iterates over a list of words; `while`/`until` loop on a command's exit status; `select` builds a numbered menu.

## For Loops

### Word List

The classic `for ... in` iterates over a space-separated list. Word splitting on the unquoted variable is what breaks `$names` into individual words here.

```bash
#!/bin/bash

names='Stan Kyle Cartman'

for name in $names; do
    echo "$name"
done

echo "All done"
```

### Brace Range

Brace expansion `{1..5}` generates a numeric sequence without needing `seq`.

```bash
#!/bin/bash

for value in {1..5}; do
    echo "$value"
done
```

### Range with a Step

Add a third number to the brace range to set the step. `{10..0..2}` counts down from 10 to 0 by 2.

```bash
#!/bin/bash

for value in {10..0..2}; do
    echo "$value"
done
```

### C-Style For

The `for (( init; test; update ))` form mirrors C and is handy when you need full control over the counter.

```bash
#!/bin/bash

for (( num = 1; num <= 5; num++ )); do
    echo "$num"
done
```

### Nested For

Nested loops combine two counters. This one-liner creates 100 files named `00` through `99`:

```bash
mkdir test && cd test
for i in 0 1 2 3 4 5 6 7 8 9; do
    for j in 0 1 2 3 4 5 6 7 8 9; do
        touch "$i$j"
    done
done
```

### Looping Over Command Output

You can iterate over a command's output, but note this splits on whitespace and expands globs. For filenames, prefer `while read` with `find -print0` (see the while-loops guide). For simple, space-free output it's fine:

```bash
#!/bin/bash

for file in $(find "$1" -maxdepth 1 -type f -name "*.sh"); do
    echo "Found script: $file"
done
```

### Looping Over an Array

Iterate values with `"${array[@]}"`, or indices with `"${!array[@]}"`. Always quote the expansion to keep elements with spaces intact.

```bash
#!/bin/bash

fruits=("apple" "banana" "cherry" "date" "elderberry")

for fruit in "${fruits[@]}"; do
    echo "Fruit: $fruit"
done

echo "Loop with index:"
for i in "${!fruits[@]}"; do
    echo "  [$i] ${fruits[$i]}"
done
```

## While and Until

### While

`while` runs its body as long as the test succeeds (exit status 0).

```bash
#!/bin/bash

counter=1
while [ $counter -le 10 ]; do
    echo "$counter"
    (( counter++ ))
done
```

### Until

`until` is the mirror image — it runs as long as the test *fails*, i.e. until it becomes true.

```bash
#!/bin/bash

counter=1
until [ $counter -gt 10 ]; do
    echo "$counter"
    (( counter++ ))
done
```

### Read a File Line by Line

`while IFS= read -r` is the safe way to read a file, preserving whitespace and backslashes.

```bash
#!/bin/bash

while IFS= read -r line; do
    echo "Line: $line"
done < "$1"
```

### Infinite Loops

Two common idioms, both exited with `break`:

```bash
#!/bin/bash

# Method 1: while true
counter=0
while true; do
    echo "Count: $counter"
    (( counter++ ))
    [ $counter -ge 5 ] && break
done

# Method 2: C-style for with no conditions
counter=0
for (( ;; )); do
    echo "Count: $counter"
    (( counter++ ))
    [ $counter -ge 5 ] && break
done
```

## The select Menu

`select` builds a numbered menu from a word list, sets the prompt from `PS3`, and re-displays until you `break`.

```bash
#!/bin/bash

names='Kyle Cartman Stan Quit'

PS3='Select character: '
select name in $names; do
    if [ "$name" == 'Quit' ]; then
        break
    fi
    echo "Hello $name"
done

echo "Bye"
```

## Loop Control: break and continue

### continue and break with a Condition

`continue` skips to the next iteration; `break` exits the loop. These examples back up files while skipping unreadable ones, or stopping early on low disk space.

```bash
#!/bin/bash
# Skip files that aren't readable

for value in "$1"/*; do
    if [ ! -r "$value" ]; then
        echo "$value not readable" 1>&2
        continue
    fi
    cp "$value" "$1/backup/"
done
```

```bash
#!/bin/bash
# Stop early if the disk is over 90% full

for value in "$1"/*; do
    used=$(df "$1" | tail -1 | awk '{ print $5 }' | sed 's/%//')
    if [ "$used" -gt 90 ]; then
        echo "Low disk space" 1>&2
        break
    fi
    cp "$value" "$1/backup/"
done
```

### break N and continue N (Nested Loops)

Give `break` or `continue` a number to act on an outer loop level. `continue 2` restarts the next iteration of the second-enclosing loop; `break 2` exits both loops.

```bash
#!/bin/bash

echo "=== continue 2 (skip to next outer iteration) ==="
for i in 1 2 3; do
    for j in A B C; do
        if [ "$j" == "B" ]; then
            continue 2
        fi
        echo "i=$i j=$j"
    done
done

echo "=== break 2 (exit both loops) ==="
for i in 1 2 3; do
    for j in A B C; do
        if [ "$i" -eq 2 ] && [ "$j" == "B" ]; then
            echo "Breaking out at i=$i j=$j"
            break 2
        fi
        echo "i=$i j=$j"
    done
done
```

## Reading Lines into an Array with mapfile

`mapfile` (alias `readarray`) reads all lines into an array in one step — often cleaner and faster than a `while read` loop when you want the whole file in memory. `-t` strips the trailing newline from each element.

```bash
#!/bin/bash

# Read a file into an array
mapfile -t lines < "$1"

echo "File has ${#lines[@]} lines"
for i in "${!lines[@]}"; do
    echo "Line $(( i + 1 )): ${lines[$i]}"
done

# readarray is an alias for mapfile
readarray -t words <<< "$(echo -e "one\ntwo\nthree")"
for word in "${words[@]}"; do
    echo "Word: $word"
done
```

## Common Practical Examples

### Transform Files in a Directory

Loop over matching files and derive an output name with `basename -s` to swap the extension.

```bash
#!/bin/bash
# Make a .php copy of each .html file in a directory

for value in "$1"/*.html; do
    cp "$value" "$1/$(basename -s .html "$value").php"
done
```

## Choosing the Right Loop

| Construct | Use when |
|-----------|----------|
| `for x in list` | You have a fixed list of words, files (globs), or array elements |
| `for (( ; ; ))` | You need a numeric counter with full control over init/test/step |
| `while cmd` | Loop while a command succeeds — reading input, polling, counters |
| `until cmd` | Loop until a command succeeds — waiting for a condition to become true |
| `select` | You want a quick interactive numbered menu |
| `mapfile` | You just need all lines of a file in an array, no per-line loop body |

## Tips

- Quote variable expansions (`"$file"`, `"${array[@]}"`) to avoid word splitting and globbing.
- Prefer `while IFS= read -r` over `for line in $(cat file)` for line-by-line reading — the `for` form splits on whitespace and expands globs.
- Use `(( ))` for numeric conditions and arithmetic; it's cleaner than `[ ]` with `-le`/`-lt`.
- `break N` / `continue N` control outer loops by level — handy for escaping deeply nested loops.
- `mapfile -t` is the fastest way to slurp a file into an array when you don't need to process lines as they're read.
