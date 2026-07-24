# Bash Test Conditions: `[ ]` vs `[[ ]]`

A guide to understanding the differences between `[ ]` (test command) and `[[ ]]` (bash keyword) for writing conditionals in shell scripts.

---

## Overview

Bash provides three forms for evaluating conditional expressions:

- **`test`** — the original POSIX command
- **`[ ]`** — a synonym for `test` (requires a closing `]`)
- **`[[ ]]`** — a bash keyword with enhanced features

`test` and `[ ]` are identical in functionality — `[` is literally the same command as `test`, just with the requirement of a closing `]` for readability. Both are POSIX-compliant. `[[ ]]` is a shell keyword (not a command), parsed differently by bash, with additional capabilities.

```bash
# These three are equivalent:
test -f /etc/passwd && echo "exists"
[ -f /etc/passwd ] && echo "exists"
[[ -f /etc/passwd ]] && echo "exists"
```

All return an exit status: 0 for true, 1 for false.

---

## Key Differences

| Feature | `[ ]` (test) | `[[ ]]` (keyword) |
|---------|:------------:|:-----------------:|
| POSIX-compliant | yes | no (bash/ksh/zsh only) |
| Type | Command (`/usr/bin/[` or built-in) | Shell keyword |
| Word splitting inside | yes (must quote variables) | no |
| Filename globbing inside | yes (must quote variables) | no |
| Pattern matching (`*`, `?`) | no | yes (with `==`) |
| Regex matching | no | yes (with `=~`) |
| Logical `&&` `||` inside | no (use `-a` `-o`) | yes |
| String comparison | `=` | `==` or `=` |
| Parentheses for grouping | `\( \)` (escaped) | `( )` (no escaping) |

---

## Why `[ ]` Requires Quoting

`[ ]` is a command, so its arguments undergo word splitting and glob expansion like any other command. An unquoted empty variable causes a syntax error:

```bash
var=""

# WRONG — expands to: [ = "hello" ] → syntax error
[ $var = "hello" ]

# CORRECT — expands to: [ "" = "hello" ]
[ "$var" = "hello" ]
```

With `[[ ]]`, the shell parses it as a keyword before word splitting, so unquoted variables are safe:

```bash
var=""

# Works fine — no word splitting inside [[ ]]
[[ $var == "hello" ]]
```

---

## String Comparisons

```bash
# Equality
[ "$a" = "$b" ]
[[ $a == $b ]]

# Inequality
[ "$a" != "$b" ]
[[ $a != $b ]]

# Less than (lexicographic) — note the escaping with [ ]
[ "$a" \< "$b" ]
[[ $a < $b ]]

# Greater than (lexicographic)
[ "$a" \> "$b" ]
[[ $a > $b ]]

# String is empty
[ -z "$var" ]
[[ -z $var ]]

# String is not empty
[ -n "$var" ]
[[ -n $var ]]
```

---

## Numeric Comparisons

Numeric operators are the same in both:

```bash
[ "$a" -eq "$b" ]       # equal
[ "$a" -ne "$b" ]       # not equal
[ "$a" -lt "$b" ]       # less than
[ "$a" -le "$b" ]       # less than or equal
[ "$a" -gt "$b" ]       # greater than
[ "$a" -ge "$b" ]       # greater than or equal
```

With `[[ ]]`, you can also use `(( ))` for arithmetic:

```bash
(( a == b ))
(( a > b ))
(( a >= b && a <= c ))
```

---

## Pattern Matching

Only available with `[[ ]]`. The right-hand side of `==` is treated as a glob pattern (do **not** quote it):

```bash
# Match files ending in .txt
[[ $file == *.txt ]]

# Match files starting with test
[[ $file == test* ]]

# Single character wildcard
[[ $char == ? ]]

# Character class
[[ $char == [a-z] ]]

# Extended globs (with shopt -s extglob)
[[ $file == +(*.jpg|*.png|*.gif) ]]
```

> If you quote the right side, it becomes a literal string comparison, not a pattern.

```bash
[[ $file == "*.txt" ]]   # Only matches the literal string "*.txt"
[[ $file == *.txt ]]     # Matches anything ending in .txt
```

---

## Regex Matching

Only available with `[[ ]]` using the `=~` operator:

```bash
# Match an email pattern
[[ $email =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]

# Match a date format (YYYY-MM-DD)
[[ $date =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]]

# Match an IP address (basic)
[[ $ip =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]]

# Capture groups with BASH_REMATCH
if [[ "2024-03-15" =~ ^([0-9]{4})-([0-9]{2})-([0-9]{2})$ ]]; then
    echo "Year: ${BASH_REMATCH[1]}"    # 2024
    echo "Month: ${BASH_REMATCH[2]}"   # 03
    echo "Day: ${BASH_REMATCH[3]}"     # 15
fi
```

> Do **not** quote the regex pattern — quoting disables regex interpretation.

---

## Logical Operators

```bash
# With [ ] — use separate commands connected by && ||
[ "$a" = "1" ] && [ "$b" = "2" ]
[ "$a" = "1" ] || [ "$b" = "2" ]

# With [ ] — using -a (AND) and -o (OR), deprecated
[ "$a" = "1" -a "$b" = "2" ]
[ "$a" = "1" -o "$b" = "2" ]

# With [[ ]] — && and || work inside
[[ $a == "1" && $b == "2" ]]
[[ $a == "1" || $b == "2" ]]

# Negation
[ ! -f "$file" ]
[[ ! -f $file ]]
```

---

## File Test Operators

These work the same in both `[ ]` and `[[ ]]`:

| Operator | Description |
|----------|-------------|
| `-e file` | File exists |
| `-f file` | Regular file exists |
| `-d file` | Directory exists |
| `-s file` | File exists and is not empty |
| `-r file` | File is readable |
| `-w file` | File is writable |
| `-x file` | File is executable |
| `-L file` | File is a symbolic link |
| `-p file` | File is a named pipe |
| `-S file` | File is a socket |
| `-b file` | File is a block device |
| `-c file` | File is a character device |
| `f1 -nt f2` | f1 is newer than f2 |
| `f1 -ot f2` | f1 is older than f2 |
| `f1 -ef f2` | f1 and f2 are the same file (hard link) |

```bash
# Check if a file exists and is readable
[[ -f /etc/passwd && -r /etc/passwd ]]

# Check if a directory exists
[[ -d /tmp/mydir ]] || mkdir /tmp/mydir

# Check if a symlink
[[ -L /usr/bin/python ]] && echo "It's a symlink"
```

---

## Common Patterns

```bash
# Check if a command exists
if [[ $(command -v git) ]]; then
    echo "git is installed"
fi

# Or with type
if type git &>/dev/null; then
    echo "git is installed"
fi

# Check if a variable is set (bash 4.2+)
[[ -v MY_VAR ]] && echo "MY_VAR is set"

# Default value if variable is empty
name=${name:-"default"}

# Check if running as root
[[ $EUID -eq 0 ]] && echo "Running as root"

# Check if string contains substring
[[ $haystack == *"needle"* ]]

# Check if argument count is correct
[[ $# -lt 2 ]] && echo "Usage: $0 <arg1> <arg2>" && exit 1
```

---

## When to Use Which

| Use case | Recommendation |
|----------|---------------|
| Bash scripts (not shared with sh/dash) | `[[ ]]` — safer, more features |
| POSIX scripts (portability required) | `[ ]` — works everywhere |
| Pattern or regex matching | `[[ ]]` — the only option |
| Dockerfiles with `/bin/sh` | `[ ]` — Alpine uses dash |
| Complex logical expressions | `[[ ]]` — cleaner syntax |
| Simple single checks | Either works |

> Rule of thumb: use `[[ ]]` in bash scripts for safer and more powerful conditionals. Use `[ ]` only when POSIX portability is required (e.g., scripts with `#!/bin/sh`).

---

## References

- [GNU Bash Manual — Conditional Expressions](https://www.gnu.org/software/bash/manual/bash.html#Bash-Conditional-Expressions)
- [BashFAQ — What is the difference between test, \[ and \[\[?](https://mywiki.wooledge.org/BashFAQ/031)
