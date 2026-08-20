# Bash Single vs Double Brackets: [ ] vs [[ ]]

Both `[ ]` and `[[ ]]` are used for conditional testing in Bash, but they have important differences in behavior, safety, and capabilities.

## What They Are

| Operator | Type | Description |
|----------|------|-------------|
| `[ ]` | POSIX test command | Built-in command that behaves like external `/bin/test` |
| `[[ ]]` | Bash keyword | Bash-specific language construct with enhanced features |

## Key Differences Summary

| Feature | `[ ]` | `[[ ]]` |
|---------|-------|---------|
| **Portability** | POSIX-compliant (works in /bin/sh) | Bash/ksh/zsh only |
| **Word Splitting** | Variables undergo splitting | No word splitting |
| **Pathname Expansion** | Variables undergo globbing | No pathname expansion |
| **Pattern Matching** | No | Yes (with `=` and `==`) |
| **Regex Support** | No | Yes (with `=~`) |
| **Logical Operators** | `-a`, `-o` (deprecated) | `&&`, `||`, `!` |
| **String Comparison** | Must escape `<` and `>` | Direct `<` and `>` |
| **Variable Testing** | Basic `-n`, `-z` | Enhanced with `-v` |
| **Quoting Requirements** | Must quote variables | Generally safe unquoted |

## Word Splitting and Globbing

### The Problem with [ ]

```bash
# DANGEROUS - unquoted variables can break
file="my file.txt"
[ -f $file ]        # ERROR: expands to [ -f my file.txt ] - too many arguments

# SAFE - quoted variables
[ -f "$file" ]      # OK: treats as single argument

# Globbing issue
pattern="*.txt"
[ -f $pattern ]     # May expand to multiple files and fail
[ -f "$pattern" ]   # OK: treats literally
```

### Safety with [[ ]]

```bash
# SAFE - no word splitting or globbing
file="my file.txt"
pattern="*.txt"
[[ -f $file ]]      # OK: no splitting
[[ -f $pattern ]]   # OK: no globbing (unless you want pattern matching)
```

## Pattern Matching

### [ ] — Literal Comparison Only

```bash
file="document.txt"
[ "$file" = "*.txt" ]     # FALSE - literal comparison with "*"
[ "$file" = "$pattern" ]  # Only true if $file literally equals $pattern
```

### [[ ]] — Glob Pattern Support

```bash
file="document.txt"
[[ $file == *.txt ]]      # TRUE - glob pattern matching
[[ $file == d*.txt ]]     # TRUE - matches pattern
[[ $file == *.pdf ]]      # FALSE - doesn't match pattern

# Case-insensitive patterns (bash 4.0+)
shopt -s nocasematch
[[ $file == *.TXT ]]      # TRUE - case insensitive
shopt -u nocasematch      # Turn off
```

## Regular Expressions

### Only Available in [[ ]]

```bash
email="user@domain.com"
phone="123-456-7890"

# Email validation
if [[ $email =~ ^[[:alnum:]._-]+@[[:alnum:].-]+\.[[:alpha:]]{2,}$ ]]; then
    echo "Valid email"
    echo "Full match: ${BASH_REMATCH[0]}"
fi

# Phone number with capture groups
if [[ $phone =~ ^([0-9]{3})-([0-9]{3})-([0-9]{4})$ ]]; then
    echo "Area code: ${BASH_REMATCH[1]}"
    echo "Exchange: ${BASH_REMATCH[2]}"
    echo "Number: ${BASH_REMATCH[3]}"
fi

# URL parsing
url="https://example.com:8080/path"
if [[ $url =~ ^(https?)://([^:/]+)(:([0-9]+))?(/.*)?$ ]]; then
    protocol=${BASH_REMATCH[1]}
    host=${BASH_REMATCH[2]}
    port=${BASH_REMATCH[4]}
    path=${BASH_REMATCH[5]}
fi
```

## String Comparison Operators

### [ ] — Must Escape Shell Operators

```bash
a="apple"
b="banana"

# Alphabetical comparison (lexicographic)
[ "$a" \< "$b" ]    # TRUE - must escape <
[ "$a" \> "$b" ]    # FALSE - must escape >

# Alternative: use external command
if [ "$(printf '%s\n' "$a" "$b" | sort | head -n1)" = "$a" ]; then
    echo "$a comes first"
fi
```

### [[ ]] — Direct Operators

```bash
a="apple"
b="banana"

# Much cleaner syntax
[[ $a < $b ]]       # TRUE - no escaping needed
[[ $a > $b ]]       # FALSE - no escaping needed

# Numeric comparison still uses same operators
[[ 10 -lt 20 ]]     # TRUE
[[ 10 < 20 ]]       # FALSE - this is string comparison!
```

## Logical Operations

### [ ] — Limited and Confusing

```bash
# Using -a and -o (DEPRECATED - avoid these)
[ -f "$file" -a -r "$file" ]    # AND - deprecated
[ -f "$file" -o -d "$file" ]    # OR - deprecated

# Better: combine separate tests
[ -f "$file" ] && [ -r "$file" ]    # AND
[ -f "$file" ] || [ -d "$file" ]    # OR

# Negation
[ ! -f "$file" ]                    # NOT
```

### [[ ]] — Clear and Powerful

```bash
# Natural logical operators
[[ -f $file && -r $file ]]          # AND
[[ -f $file || -d $file ]]          # OR
[[ ! -f $file ]]                    # NOT

# Grouping with parentheses
[[ ( -f $file && -r $file ) || -d $file ]]

# Complex conditions
[[ $user == "admin" || ( $user == "user" && $group == "wheel" ) ]]
```

## Variable Testing

### [ ] — Basic Tests

```bash
# Standard tests
[ -n "$var" ]       # True if var is non-empty
[ -z "$var" ]       # True if var is empty
[ "$var" ]          # True if var is non-empty (shorthand)
```

### [[ ]] — Enhanced Tests

```bash
# All the above plus:
[[ -v var ]]        # True if variable is SET (bash 4.2+)
[[ -v arr[5] ]]    # True if array element exists

# Examples
unset myvar
[[ -v myvar ]]      # FALSE - variable not set
[[ -z $myvar ]]     # TRUE - variable is empty

myvar=""
[[ -v myvar ]]      # TRUE - variable is set
[[ -z $myvar ]]     # TRUE - variable is empty

declare -a arr=(a b c)
[[ -v arr[1] ]]     # TRUE - element exists
[[ -v arr[5] ]]     # FALSE - element doesn't exist
```

## File Tests (Same in Both)

```bash
# File existence and type
[ -e "$file" ]      # Exists
[ -f "$file" ]      # Regular file
[ -d "$dir" ]       # Directory
[ -L "$link" ]      # Symbolic link
[ -s "$file" ]      # Non-empty file

# Permissions
[ -r "$file" ]      # Readable
[ -w "$file" ]      # Writable
[ -x "$file" ]      # Executable

# File comparisons
[ "$file1" -nt "$file2" ]   # Newer than
[ "$file1" -ot "$file2" ]   # Older than
[ "$file1" -ef "$file2" ]   # Same file (hardlink)
```

## Numeric Comparisons (Same in Both)

```bash
# Arithmetic comparisons
[ "$a" -eq "$b" ]   # Equal
[ "$a" -ne "$b" ]   # Not equal
[ "$a" -lt "$b" ]   # Less than
[ "$a" -le "$b" ]   # Less than or equal
[ "$a" -gt "$b" ]   # Greater than
[ "$a" -ge "$b" ]   # Greater than or equal

# Examples
age=25
[[ $age -ge 18 && $age -lt 65 ]]    # Working age range
[ "$age" -ge 18 ] && [ "$age" -lt 65 ]  # POSIX equivalent
```

## Practical Examples

### Input Validation

```bash
#!/bin/bash

validate_email() {
    local email="$1"

    # Bash version with regex
    if [[ $email =~ ^[[:alnum:]._-]+@[[:alnum:].-]+\.[[:alpha:]]{2,}$ ]]; then
        return 0
    else
        return 1
    fi
}

validate_file() {
    local file="$1"

    # Multiple conditions with [[ ]]
    if [[ -f $file && -r $file && -s $file ]]; then
        echo "File is readable and non-empty"
        return 0
    elif [[ -f $file ]]; then
        echo "File exists but may be empty or unreadable"
        return 1
    else
        echo "File does not exist"
        return 2
    fi
}
```

### Configuration Checking

```bash
#!/bin/bash

check_config() {
    local config_file="$1"

    # Check if config file exists and is readable
    [[ -f $config_file && -r $config_file ]] || {
        echo "Config file not found or not readable"
        return 1
    }

    # Read config values
    while IFS='=' read -r key value; do
        # Skip comments and empty lines
        [[ $key =~ ^#.*$ || -z $key ]] && continue

        case $key in
            "DEBUG")
                [[ $value =~ ^(true|false)$ ]] || {
                    echo "Invalid DEBUG value: $value"
                    return 1
                }
                ;;
            "PORT")
                [[ $value =~ ^[0-9]+$ && $value -gt 0 && $value -le 65535 ]] || {
                    echo "Invalid PORT value: $value"
                    return 1
                }
                ;;
        esac
    done < "$config_file"
}
```

## Best Practices

### When to Use [ ]

```bash
#!/bin/sh
# POSIX shell script - must use [ ]

if [ -f "$config_file" ]; then
    echo "Config found"
fi

# Always quote variables
if [ -n "$USER" ] && [ "$USER" = "root" ]; then
    echo "Running as root"
fi
```

### When to Use [[ ]]

```bash
#!/bin/bash
# Bash script - prefer [[ ]]

# Pattern matching
if [[ $filename == *.log ]]; then
    process_log_file "$filename"
fi

# Regex validation
if [[ $input =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]]; then
    echo "Valid date format"
fi

# Complex logic
if [[ -f $file && ( $force == true || ! -f $backup ) ]]; then
    cp "$file" "$backup"
fi
```

## Common Pitfalls

### Unquoted Variables in [ ]

```bash
# WRONG
name="John Doe"
[ -n $name ]        # ERROR: [ -n John Doe ] - too many arguments

# CORRECT
[ -n "$name" ]      # OK: [ -n "John Doe" ]
[[ -n $name ]]      # OK: no word splitting in [[ ]]
```

### Pattern Matching Confusion

```bash
# WRONG - expecting pattern match in [ ]
[[ $file == *.txt ]]     # Pattern match - CORRECT
[ "$file" = *.txt ]      # Literal comparison - likely WRONG

# WRONG - expecting literal match in [[ ]]
[[ $file == "*.txt" ]]   # Literal comparison - correct if intended
[[ $file == *.txt ]]     # Pattern match - correct if intended
```

### Regex Quoting

```bash
# WRONG - quoted regex becomes literal
pattern="[0-9]+"
[[ $input =~ "$pattern" ]]   # Treats as literal string "[0-9]+"

# CORRECT - unquoted for regex
[[ $input =~ [0-9]+ ]]       # Treats as regex
[[ $input =~ $pattern ]]     # Variable expansion, then regex
```

## Arithmetic Tests with (( ))

For numeric comparisons, Bash offers `(( ))` which is cleaner than `-eq`/`-lt` inside brackets:

```bash
# Standard bracket syntax
[[ $count -gt 0 && $count -le 100 ]]
[ "$count" -gt 0 ] && [ "$count" -le 100 ]

# Arithmetic syntax - more natural for numbers
(( count > 0 && count <= 100 ))

# Supports C-style operators
(( x == 5 ))        # Equal
(( x != 5 ))        # Not equal
(( x > 5 ))         # Greater than
(( x >= 5 ))        # Greater than or equal
(( x < 5 ))         # Less than
(( x <= 5 ))        # Less than or equal

# Arithmetic expressions
(( result = x + y ))
(( x++ ))
(( x-- ))
(( x += 10 ))

# Use in if statements
if (( age >= 18 && age < 65 )); then
    echo "Working age"
fi

# Loop with arithmetic
for (( i = 0; i < 10; i++ )); do
    echo "$i"
done
```

### When to Use (( )) vs [[ ]] for Numbers

```bash
# Prefer (( )) for pure arithmetic
(( total = price * quantity ))
(( discount > 0 )) && (( total -= discount ))

# Use [[ ]] when mixing string and numeric tests
[[ -f $file && $(wc -l < "$file") -gt 100 ]]

# CAUTION: (( )) treats unset/empty variables as 0
unset x
(( x == 0 ))       # TRUE - unset treated as 0
[[ $x -eq 0 ]]     # ERROR in set -u (unbound variable)
```

## Performance

`[[ ]]` is a Bash keyword parsed at shell level, while `[ ]` is a builtin command that still goes through command lookup. In tight loops, this matters:

```bash
# Benchmark: 100,000 iterations
# [ ] - builtin command (slower)
time for i in $(seq 1 100000); do [ "$i" -gt 50000 ]; done
# real ~2.5s

# [[ ]] - shell keyword (faster)
time for i in $(seq 1 100000); do [[ $i -gt 50000 ]]; done
# real ~1.8s

# (( )) - arithmetic keyword (fastest for numbers)
time for i in $(seq 1 100000); do (( i > 50000 )); done
# real ~1.5s
```

> **Note:** For typical scripts with a few conditionals, the difference is negligible. It only matters in loops running thousands of iterations.

## Quick Reference

### Choose [ ] when:

- Writing POSIX-compliant scripts (/bin/sh)
- Maximum portability is required
- Simple comparisons only

### Choose [[ ]] when:

- Writing Bash-specific scripts
- Need pattern matching or regex
- Complex logical conditions
- Want safer variable handling

### Remember:

- Always quote variables in `[ ]`
- Generally safe to leave unquoted in `[[ ]]`
- Use regex with `=~` only in `[[ ]]`
- Use `&&`, `||` inside `[[ ]]`, combine separate `[ ]` with shell `&&`, `||`
