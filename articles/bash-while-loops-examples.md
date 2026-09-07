# Bash While Loop Examples

A collection of practical `while` loop patterns in Bash — counters, reading files line by line, field splitting with `IFS`, retry logic, nested loops, and the input-feeding techniques (redirection, pipes, here strings, and process substitution) that make `while read` reliable.

> The `while` loop runs its body as long as the test command succeeds (exit status 0). The test can be a `[ ]`/`[[ ]]` condition, an arithmetic `(( ))` expression, or any command — including `read`, which succeeds until it hits end of input.

## Counters and Conditions

### Simple Counter (one-liner)

A `while` loop using `[[ ]]` and arithmetic increment to print a message 10 times. Initialize the variable first so the test isn't comparing against an empty value.

```bash
NUMBER=1
while [[ $NUMBER -le 10 ]]; do echo "Welcome ${NUMBER} times"; (( NUMBER++ )); done
```

### Arithmetic Condition

Double parentheses `(( ))` let you write the condition with natural operators (`<=`, `<`, `>`) and avoid `[` / `test` entirely.

```bash
#!/bin/bash

count=1
while (( count <= 10 )); do
    echo "Count: $count"
    (( count++ ))
done

echo "All done"
```

### Multiple Conditions

Combine tests with `&&` (or `||`) between separate `[ ]` commands. Keep each condition in its own brackets.

```bash
#!/bin/bash

count=0
max=10
found=false

while [ $count -lt $max ] && [ "$found" == "false" ]; do
    echo "Checking item $count..."
    if [ $count -eq 7 ]; then
        found=true
        echo "Found at $count!"
    fi
    (( count++ ))
done
```

### Infinite Loop with Break

An infinite `while true` loop that reads input and exits when a sentinel value is entered.

```bash
#!/bin/bash

while true; do
    read -p "Enter a command (quit to exit): " cmd
    if [ "$cmd" == "quit" ]; then
        echo "Goodbye!"
        break
    fi
    echo "You entered: $cmd"
done
```

## Reading Files and Input

### Read a File Line by Line

The simplest form of `while read` with input redirection. Use `IFS=` and `read -r` to preserve leading/trailing whitespace and backslashes.

```bash
#!/bin/bash

while IFS= read -r myline; do
    echo "$myline"
done < inputfile
```

### Read with Line Numbers

Track line numbers manually while reading — useful for log parsing or error reporting. This example skips comment lines starting with `#`.

```bash
#!/bin/bash

lineno=0
while IFS= read -r line; do
    (( lineno++ ))
    if [[ "$line" =~ ^# ]]; then
        continue
    fi
    echo "$lineno: $line"
done < "$1"

echo "Processed $lineno lines"
```

### Here String Input

Feed a variable straight into a `while read` loop with a here string (`<<<`). Handy when the data is already in a variable.

```bash
#!/bin/bash

data="server1:active
server2:down
server3:active
server4:maintenance"

while IFS=":" read -r host status; do
    echo "Host: $host -> Status: $status"
done <<< "$data"
```

### NUL-Delimited Input (find -print0)

Filenames can contain spaces and even newlines, so splitting on lines is unsafe. Pair `find -print0` with `read -d ''` (an empty delimiter means NUL) to loop over filenames safely.

```bash
#!/bin/bash

while IFS= read -r -d '' file; do
    echo "Processing: $file"
done < <(find . -type f -name '*.log' -print0)
```

## Field Splitting with IFS

### Parse /etc/passwd (redirection)

Read `/etc/passwd` line by line and split each colon-delimited line into named fields with `IFS=":"`.

```bash
#!/bin/bash

while IFS=":" read -r user password uid gid gecos home shell; do
    echo "$gecos"
done < /etc/passwd
```

### Parse a CSV File

Set `IFS=","` inline with `read` to split each line into named variables.

```bash
#!/bin/bash

while IFS="," read -r name age city; do
    echo "Name: $name | Age: $age | City: $city"
done < contacts.csv
```

### Extracting Fields with cut

An alternative when you want specific fields without listing all of them — pipe each line to `cut`. Slower (spawns a process per line), but readable for quick jobs.

```bash
#!/bin/bash

while IFS= read -r inputline; do
    login="$(echo "$inputline" | cut -d: -f1)"
    fulln="$(echo "$inputline" | cut -d: -f4)"
    echo "login = $login and fullname = $fulln"
done < /etc/passwd
```

## Feeding the Loop: Pipes vs Process Substitution

### The Pipe Subshell Pitfall

Piping into `while` runs the loop in a **subshell**, so variables modified inside the loop are lost when it ends:

```bash
# Beware: $count is 0 after this loop, not the real total
cat /etc/passwd | while IFS=":" read -r user rest; do
    (( count++ ))
done
echo "$count"   # prints 0 — the loop ran in a subshell
```

### Process Substitution (preserves variables)

Use `< <(command)` instead of a pipe so the loop runs in the current shell and variable changes stick.

```bash
#!/bin/bash

total=0
while read -r size file; do
    echo "$file: $size bytes"
    (( total += size ))
done < <(du -b /tmp/*)

echo "Total: $total bytes"
```

### for line in $(< file) — Not Recommended

You may see `for line in $(< file.txt)` used to read a file, but it splits on **whitespace**, not newlines, and expands globs. Prefer `while IFS= read -r` for line-by-line reading.

```bash
# Splits on spaces/tabs too, not just newlines — usually not what you want
for line in $(< file.txt); do
    echo "$line"
done
```

## Practical Patterns

### SSH Loop from a Host File

Read host entries from a file and run a remote command per host. Use `ssh -n` so SSH doesn't consume the loop's stdin (which would break `read`).

```bash
#!/bin/bash

while read -r host; do
    ssh -n "user@$host" "ls -l"
done < hosts.txt
```

### Nested Loops

Nested `while` loops with independent counters, here building a grid.

```bash
#!/bin/bash

row=1
while [ $row -le 3 ]; do
    col=1
    line=""
    while [ $col -le 5 ]; do
        line+="[$row,$col] "
        (( col++ ))
    done
    echo "$line"
    (( row++ ))
done
```

### Parsing Options with getopts

The standard argument-parsing idiom is a `while getopts` loop. Each pass reads the next option; a trailing `:` in the option string (e.g. `o:`) means that option takes an argument, delivered in `$OPTARG`.

```bash
#!/bin/bash

verbose=false
output=""

while getopts "vo:h" opt; do
    case "$opt" in
        v) verbose=true ;;
        o) output="$OPTARG" ;;
        h) echo "Usage: $0 [-v] [-o file]"; exit 0 ;;
        *) echo "Invalid option" >&2; exit 1 ;;
    esac
done
shift $(( OPTIND - 1 ))   # drop the parsed options, leaving positional args

echo "verbose=$verbose output=$output remaining=$*"
```

### Commands That Consume stdin

Some commands (`ssh`, `mysql`, `ffmpeg`) read from stdin by default and will swallow the rest of your loop's input, causing it to run only once. Redirect their input from `/dev/null` (or use `ssh -n`) to protect the loop.

```bash
#!/bin/bash

while IFS= read -r host; do
    ssh "user@$host" "uptime" </dev/null    # </dev/null keeps ssh off the loop's stdin
done < hosts.txt
```

### Retry Loop with Sleep

Retry a command up to a maximum number of attempts with a delay between tries. Exits non-zero if all attempts fail.

```bash
#!/bin/bash

max_retries=5
attempt=1

while [ $attempt -le $max_retries ]; do
    echo "Attempt $attempt of $max_retries..."
    if ping -c 1 -W 2 google.com &>/dev/null; then
        echo "Success on attempt $attempt"
        break
    fi
    echo "Failed. Retrying in 3 seconds..."
    sleep 3
    (( attempt++ ))
done

if [ $attempt -gt $max_retries ]; then
    echo "All $max_retries attempts failed."
    exit 1
fi
```

### Waiting and Polling

Loop until a condition becomes true — commonly "wait until a service is accepting connections". The negated test (`while ! ...`) keeps looping while the command fails.

```bash
#!/bin/bash

# Wait for a TCP port to open (e.g. a database starting up)
until nc -z localhost 5432; do
    echo "Waiting for postgres..."
    sleep 1
done
echo "Postgres is up"
```

Add a bound so it can't wait forever:

```bash
#!/bin/bash

deadline=$(( SECONDS + 30 ))   # give up after 30 seconds
while ! nc -z localhost 5432; do
    if (( SECONDS >= deadline )); then
        echo "Timed out waiting for postgres" >&2
        exit 1
    fi
    sleep 1
done
echo "Postgres is up"
```

## Tips

- Always use `read -r` unless you specifically want backslash escapes interpreted.
- Set `IFS=` (empty) on the `read` line to preserve leading and trailing whitespace; set `IFS=":"` or `IFS=","` to split on a delimiter.
- Prefer input redirection (`< file`) or process substitution (`< <(cmd)`) over piping into `while` when the loop needs to update variables in the current shell.
- Quote your variables (`"$line"`, `"$host"`) to avoid word splitting and globbing.
- Use `(( ))` for numeric conditions and increments — it's cleaner than `[ ]` with `-le`/`-lt`.
