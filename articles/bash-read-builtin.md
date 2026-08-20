# Bash Read Builtin: Examples and Patterns

A collection of examples demonstrating various uses of the `read` builtin in Bash.

## Examples

### Basic Read with Comparison

Reads three numbers from the user and determines the greatest using nested `if` statements.

```bash
#!/bin/bash
echo "Enter num1:"
read n1
echo "Enter num2:"
read n2
echo "Enter num3:"
read n3

if [ $n1 -gt $n2 ]; then
    if [ $n1 -gt $n3 ]; then
        echo "Greatest among the three is $n1"
    else
        echo "Greatest among the three is $n3"
    fi
else
    if [ $n2 -gt $n3 ]; then
        echo "Greatest among the three is $n2"
    else
        echo "Greatest among the three is $n3"
    fi
fi
exit 0
```

### Read with Inline Prompt (-p)

Uses `read -p` to display a prompt and capture input on the same line.

```bash
#!/bin/bash

read -p "Enter your name: " name
read -p "Enter your age: " age

echo "Hello $name, you are $age years old."
```

### Silent Read for Passwords (-s)

Uses `read -s` to suppress input display, useful for passwords and secrets.

```bash
#!/bin/bash

read -p "Username: " username
read -sp "Password: " password
echo ""

echo "Authenticating $username..."
```

### Read with Timeout (-t)

Uses `read -t` to set a timeout in seconds. If the user doesn't respond in time, the script continues with a default value.

```bash
#!/bin/bash

read -t 5 -p "Enter your environment (5s timeout) [dev]: " env
env="${env:-dev}"

echo "Using environment: $env"
```

### Read Single Character (-n)

Uses `read -n 1` to capture a single keypress without requiring Enter.

```bash
#!/bin/bash

read -n 1 -p "Continue? (y/n): " answer
echo ""

case $answer in
    y|Y) echo "Continuing..." ;;
    n|N) echo "Aborting."; exit 1 ;;
    *) echo "Invalid input."; exit 1 ;;
esac
```

### Read into Multiple Variables

Splits input into multiple variables based on whitespace. Any leftover words go into the last variable.

```bash
#!/bin/bash

echo "Enter first name, last name, and city:"
read first last city

echo "First: $first"
echo "Last: $last"
echo "City: $city"
```

### Read with Custom Delimiter (-d)

Uses `read -d` to set a custom delimiter instead of newline. Input ends when the delimiter character is typed.

```bash
#!/bin/bash

echo "Enter values separated by commas (end with ;):"
read -d ";" input

echo ""
echo "You entered: $input"
```

### Read into an Array (-a)

Uses `read -a` to split input into an array by whitespace.

```bash
#!/bin/bash

read -p "Enter fruits separated by spaces: " -a fruits

echo "You entered ${#fruits[@]} fruits:"
for i in "${!fruits[@]}"; do
    echo "  [$i] ${fruits[$i]}"
done
```

### Read a File Line by Line

Redirects a file into a `while read` loop using `IFS=` and `-r` to preserve whitespace and backslashes.

```bash
#!/bin/bash

while IFS= read -r line; do
    echo ">> $line"
done < "$1"

echo "Done reading file."
```

### Read from a Here String

Uses a here string (`<<<`) to feed a value directly into `read` with a custom `IFS` for splitting.

```bash
#!/bin/bash

data="john:28:developer"

IFS=: read -r name age role <<< "$data"

echo "Name: $name"
echo "Age: $age"
echo "Role: $role"
```

### Read with Character Limit and Timeout Combined

Combines `-n`, `-t`, and `-s` for a countdown-style confirmation prompt.

```bash
#!/bin/bash

echo "Press any key within 3 seconds to cancel deployment..."
if read -t 3 -n 1 -s; then
    echo ""
    echo "Deployment cancelled."
    exit 1
fi

echo ""
echo "Deploying..."
```

### Read with Input Validation Loop

Loops until the user provides valid input, using `read` inside a `while` loop with regex validation.

```bash
#!/bin/bash

while true; do
    read -p "Enter an email address: " email
    if [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
        break
    fi
    echo "Invalid email. Try again."
done

echo "Email accepted: $email"
```

### Read with Default Value

Prompts the user and falls back to a default using `${VAR:-default}` when input is left blank.

```bash
#!/bin/bash

read -p "Hostname [localhost]: " host
read -p "Port [8080]: " port
read -p "Protocol [https]: " protocol

host="${host:-localhost}"
port="${port:-8080}"
protocol="${protocol:-https}"

echo "Connecting to $protocol://$host:$port"
```

## Reading Files

Simple, practical patterns for reading file content in Bash.

### Read Entire File into a Variable

```bash
#!/bin/bash
content=$(< "$1")
echo "$content"
```

### Read File Line by Line with Line Numbers

```bash
#!/bin/bash
n=1
while IFS= read -r line; do
    echo "$n: $line"
    ((n++))
done < "$1"
```

### Read Only the First N Lines

```bash
#!/bin/bash
N="${2:-10}"
count=0
while IFS= read -r line && (( count < N )); do
    echo "$line"
    ((count++))
done < "$1"
```

### Read a Specific Line by Number

```bash
#!/bin/bash
FILE="$1"
LINE_NUM="$2"

n=0
while IFS= read -r line; do
    ((n++))
    if (( n == LINE_NUM )); then
        echo "$line"
        break
    fi
done < "$FILE"
```

### Skip Header and Process Remaining Lines

```bash
#!/bin/bash
{
    read -r header  # consume and discard header
    while IFS= read -r line; do
        echo "Data: $line"
    done
} < "$1"
```

### Read CSV File (Comma-Separated)

```bash
#!/bin/bash
while IFS=',' read -r name age city; do
    echo "Name=$name  Age=$age  City=$city"
done < "$1"
```

### Read Colon-Delimited File (e.g., /etc/passwd)

```bash
#!/bin/bash
while IFS=: read -r user _ uid gid _ home shell; do
    echo "$user (uid=$uid) -> $shell"
done < /etc/passwd
```

### Read Tab-Separated File

```bash
#!/bin/bash
while IFS=$'\t' read -r col1 col2 col3; do
    echo "[$col1] [$col2] [$col3]"
done < "$1"
```

### Read File into an Array (One Line per Element)

```bash
#!/bin/bash
mapfile -t lines < "$1"

echo "Total lines: ${#lines[@]}"
for i in "${!lines[@]}"; do
    echo "  [$i] ${lines[$i]}"
done
```

### Read File and Filter Lines with a Pattern

```bash
#!/bin/bash
while IFS= read -r line; do
    [[ "$line" =~ ^#  ]] && continue   # skip comments
    [[ -z "$line" ]] && continue       # skip empty lines
    echo "$line"
done < "$1"
```

### Read File and Count Matching Lines

```bash
#!/bin/bash
count=0
while IFS= read -r line; do
    [[ "$line" == *"$2"* ]] && ((count++))
done < "$1"
echo "Found $count lines containing '$2'"
```

### Read Two Files Side by Side (paste-style)

```bash
#!/bin/bash
exec 3< "$1"
exec 4< "$2"

while IFS= read -r left <&3 && IFS= read -r right <&4; do
    echo "$left | $right"
done

exec 3<&-
exec 4<&-
```

### Read File Backwards (Last Line First)

```bash
#!/bin/bash
mapfile -t lines < "$1"
for (( i=${#lines[@]}-1; i>=0; i-- )); do
    echo "${lines[$i]}"
done
```

### Read a Config File (key=value)

```bash
#!/bin/bash
declare -A config

while IFS= read -r line; do
    [[ "$line" =~ ^#  ]] && continue
    [[ -z "$line" ]] && continue
    key="${line%%=*}"
    val="${line#*=}"
    config["$key"]="$val"
done < "$1"

echo "host = ${config[host]}"
echo "port = ${config[port]}"
```

### Read JSON-like Lines and Extract a Field

```bash
#!/bin/bash
# Simple line-by-line extraction (not a real JSON parser)
while IFS= read -r line; do
    if [[ "$line" =~ \"name\":[[:space:]]*\"([^\"]+)\" ]]; then
        echo "Name: ${BASH_REMATCH[1]}"
    fi
done < "$1"
```

### Read from Command Output (Process Substitution)

```bash
#!/bin/bash
while IFS= read -r line; do
    echo "File: $line"
done < <(find /var/log -name "*.log" -maxdepth 1 2>/dev/null)
```

### Read from a Pipe (stdin)

```bash
#!/bin/bash
# Usage: cat file.txt | ./script.sh
while IFS= read -r line; do
    echo ">> $line"
done
```

### Read File and Build a Comma-Separated String

```bash
#!/bin/bash
result=""
while IFS= read -r line; do
    [[ -z "$line" ]] && continue
    result="${result:+$result,}$line"
done < "$1"
echo "$result"
```

### Read File with a Progress Counter

```bash
#!/bin/bash
total=$(wc -l < "$1")
n=0
while IFS= read -r line; do
    ((n++))
    printf "\rProcessing %d/%d..." "$n" "$total"
    # ... do work with $line ...
done < "$1"
echo ""
echo "Done."
```

### Read Only Non-Empty, Non-Comment Lines (Clean Config Reader)

```bash
#!/bin/bash
while IFS= read -r line; do
    line="${line%%#*}"       # strip inline comments
    line="${line%"${line##*[![:space:]]}"}"  # trim trailing whitespace
    [[ -z "$line" ]] && continue
    echo "$line"
done < "$1"
```

### Read a File and Write Transformed Output

```bash
#!/bin/bash
while IFS= read -r line; do
    echo "${line^^}"        # uppercase every line
done < "$1" > output.txt
echo "Uppercased version written to output.txt"
```

### Read Fixed-Width Fields

```bash
#!/bin/bash
# Example: first 10 chars = name, next 5 = id, rest = description
while IFS= read -r line; do
    name="${line:0:10}"
    id="${line:10:5}"
    desc="${line:15}"
    echo "Name=[${name}] ID=[${id}] Desc=[${desc}]"
done < "$1"
```

### Read a File with a Custom Record Separator (Null-Delimited)

```bash
#!/bin/bash
# Useful for filenames with spaces/newlines (e.g., from find -print0)
while IFS= read -r -d '' file; do
    echo "Found: $file"
done < <(find . -name "*.sh" -print0)
```

## Quick Reference

| Flag | Purpose | Example |
|------|---------|---------|
| `-p` | Inline prompt | `read -p "Name: " name` |
| `-s` | Silent (no echo) | `read -sp "Password: " pass` |
| `-t N` | Timeout in seconds | `read -t 5 input` |
| `-n N` | Read N characters | `read -n 1 char` |
| `-d X` | Custom delimiter | `read -d ";" input` |
| `-a` | Read into array | `read -a arr` |
| `-r` | No backslash escaping | `read -r line` |
| `IFS=` | Preserve leading whitespace | `IFS= read -r line` |
