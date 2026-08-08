<p align="center">
  <img src="/articles/images/bash-logo.svg" alt="bash logo" width="200"/>
</p>

<h1 align="center">Bash Cheatsheet</h1>

## Quoting and Escaping

Three quoting types control how Bash parses your input:

| Type | Syntax | Behavior |
|------|--------|----------|
| Per-character escaping | `\x` | Escapes the next character's special meaning |
| Weak quoting | `"..."` | Allows variable/command expansion, prevents word splitting and globbing |
| Strong quoting | `'...'` | Everything is literal, no expansion at all |

### Per-character Escaping (\)

```bash
echo \$HOME          # prints: $HOME (literal, not expanded)
echo \"hello\"       # prints: "hello" (literal quotes)
echo it\'s           # prints: it's

# Line continuation (split long commands across lines)
echo "this is" \
     "one command"
```

### Weak Quoting (Double Quotes "...")

```bash
# Variable expansion WORKS inside double quotes
echo "Your PATH is: $PATH"

# Word splitting and globbing are PREVENTED
ls -l "my file with spaces.txt"
ls -l "*"  # literal asterisk, not a glob

# Backslash only special before: $ ` " \ newline
echo "Price is \$5"   # prints: Price is $5
echo "Path is \x"     # prints: Path is \x (backslash preserved)
```

### Strong Quoting (Single Quotes '...')

```bash
# NOTHING is interpreted inside single quotes
echo 'Your PATH is: $PATH'  # prints literally: Your PATH is: $PATH
echo '$HOME `whoami` $((1+1))'  # all literal

# To include a single quote in a single-quoted string:
echo 'Here'\''s my test'     # break out and re-enter
echo "Here's my test"        # or just use double quotes
```

### ANSI-C Quoting ($'...')

```bash
# Interprets escape sequences like C
echo $'Line1\nLine2\tTabbed'
echo $'Alert: \a'
echo $'Hex char: \x41'      # prints: A
echo $'Unicode: \u2764'     # prints: ❤
```

| Code | Meaning |
|------|---------|
| `\\` | Backslash |
| `\'` | Single quote |
| `\"` | Double quote |
| `\n` | Newline |
| `\t` | Horizontal tab |
| `\r` | Carriage return |
| `\a` | Alert (bell) |
| `\b` | Backspace |
| `\e` or `\E` | Escape (ASCII 033) |
| `\f` | Form feed |
| `\v` | Vertical tab |
| `\xHH` | Hex byte |
| `\uXXXX` | Unicode (4 hex digits, Bash 4.2+) |
| `\UXXXXXXXX` | Unicode (8 hex digits, Bash 4.2+) |
| `\nnn` | Octal byte |
| `\cx` | Control-x character |

## Expansions and Substitutions

Bash processes expansions in this order (first to last):

1. **Brace expansion** `{a,b,c}` `{1..10}`
2. **Tilde expansion** `~` `~user`
3. **Parameter expansion** `$var` `${var}`  
   **Arithmetic expansion** `$((expr))`  
   **Command substitution** `$(cmd)`  
   *(these three happen simultaneously, left to right)*
4. **Word splitting**
5. **Pathname expansion** (globbing)

Process substitution `<(cmd)` `>(cmd)` happens alongside step 3.

### Tilde Expansion

| Syntax | Expands to |
|--------|-----------|
| `~` | `$HOME` |
| `~user` | Home directory of `user` |
| `~+` | `$PWD` (current directory) |
| `~-` | `$OLDPWD` (previous directory) |

### Command Substitution

```bash
# Modern syntax (preferred, nestable)
result=$(command arg1 arg2)
files=$(find . -name "*.txt")

# Legacy syntax (harder to nest)
result=`command arg1 arg2`
```

### Process Substitution

```bash
# Treat command output as a file
diff <(sort file1) <(sort file2)

# Feed command output as stdin via file descriptor
while read -r line; do
    echo "$line"
done < <(find . -type f)

# Write to a process (less common)
tee >(gzip > file.gz) > plain.txt
```

### Word Splitting

Word splitting occurs on unquoted expansions using characters in `$IFS` (default: space, tab, newline).

```bash
# Word splitting happens:
files="one two three"
for f in $files; do echo "$f"; done   # 3 iterations

# Word splitting prevented:
for f in "$files"; do echo "$f"; done  # 1 iteration

# Custom IFS for splitting
IFS=':' read -ra dirs <<< "$PATH"
```

### Pathname Expansion (Globbing)

```bash
echo *.txt            # all .txt files
echo /data/*-av/*.mp? # glob with wildcards
echo file[0-9].log    # file0.log through file9.log
```

## Parameter Expansion

### Basic Forms

| Syntax | Description | Example |
|--------|-------------|---------|
| `$var` | Simple expansion | `echo $HOME` |
| `${var}` | Explicit expansion (avoids ambiguity) | `echo "${var}s"` |
| `${!var}` | Indirection (expand variable named by var's value) | `x=HOME; echo ${!x}` |

### String Length

```bash
str="Hello World"
echo ${#str}       # 11
```

### Substring Expansion

```bash
str="Hello World"
echo ${str:6}      # World (offset 6 to end)
echo ${str:0:5}    # Hello (offset 0, length 5)
echo ${str: -5}    # World (last 5 chars, note the space!)
echo ${str:0:-6}   # Hello (from start, exclude last 6)
```

### Substring Removal (Pattern Trimming)

| Syntax | Description | Example |
|--------|-------------|---------|
| `${var#pattern}` | Remove shortest match from beginning | `${file#*.}` → removes up to first dot |
| `${var##pattern}` | Remove longest match from beginning | `${file##*.}` → get extension |
| `${var%pattern}` | Remove shortest match from end | `${file%.*}` → remove extension |
| `${var%%pattern}` | Remove longest match from end | `${path%%/*}` → get first component |

```bash
file="/home/user/document.tar.gz"

echo ${file##*/}    # document.tar.gz  (filename)
echo ${file%/*}     # /home/user       (directory)
echo ${file##*.}    # gz               (extension)
echo ${file%%.*}    # /home/user/document  (remove all extensions... wait)
# Actually: /home/user/document  -- no, let's be precise:
echo ${file%.*}     # /home/user/document.tar  (remove last extension)
echo ${file##*.}    # gz  (get last extension)
```

### Search and Replace

| Syntax | Description |
|--------|-------------|
| `${var/pattern/string}` | Replace first match |
| `${var//pattern/string}` | Replace all matches |
| `${var/#pattern/string}` | Replace if matches at beginning |
| `${var/%pattern/string}` | Replace if matches at end |

```bash
str="hello world hello"
echo ${str/hello/hi}      # hi world hello  (first only)
echo ${str//hello/hi}     # hi world hi     (all)
echo ${str/#hello/hi}     # hi world hello  (anchored at start)
echo ${str/%hello/hi}     # hello world hi  (anchored at end)

# Remove matches (omit replacement)
echo ${str//hello}        # " world "
```

### Case Modification (Bash 4+)

| Syntax | Description |
|--------|-------------|
| `${var^}` | Uppercase first character |
| `${var^^}` | Uppercase all characters |
| `${var,}` | Lowercase first character |
| `${var,,}` | Lowercase all characters |

```bash
name="john smith"
echo ${name^}     # John smith
echo ${name^^}    # JOHN SMITH

upper="HELLO"
echo ${upper,}    # hELLO
echo ${upper,,}   # hello

# Rename files to lowercase
for file in *.TXT; do mv "$file" "${file,,}"; done
```

### Default Values

| Syntax | Description |
|--------|-------------|
| `${var:-default}` | Use default if var is unset or empty |
| `${var-default}` | Use default if var is unset only |
| `${var:=default}` | Assign default if var is unset or empty |
| `${var=default}` | Assign default if var is unset only |
| `${var:+alternate}` | Use alternate if var IS set and non-empty |
| `${var+alternate}` | Use alternate if var IS set |
| `${var:?error msg}` | Error and exit if var is unset or empty |
| `${var?error msg}` | Error and exit if var is unset |

The difference between `:-` and `:=` is what happens to the variable afterward:

**`${var:-default}`** — Uses "default" for this expansion only. The variable stays unset/empty.

```bash
unset name
echo "Hello ${name:-World}"   # prints: Hello World
echo "name is: $name"         # prints: name is:  (still empty!)
```

**`${var:=default}`** — Uses "default" AND assigns it to the variable permanently.

```bash
unset name
echo "Hello ${name:=World}"   # prints: Hello World
echo "name is: $name"         # prints: name is: World (now set!)
```

In short: `:-` is a one-time fallback, `:=` is "set it if missing."

The most common real-world pattern for `:=` is:

```bash
: "${TIMEOUT:=30}"      # ensure TIMEOUT has a value for the rest of the script
: "${LOG_DIR:=/var/log}"
```

The `:` (colon/true) command discards the expansion result — you only want the side effect of assigning the variable.

```bash
# Error if unset
: ${CONFIG_FILE:?"CONFIG_FILE must be set"}

# Alternate value (for checking if set)
if [[ ${var+isset} == isset ]]; then
    echo "var is set (maybe empty)"
fi
```

## Arrays

### Indexed Arrays

```bash
# Declaration
declare -a fruits=("apple" "banana" "cherry")
files=(*.txt)                    # glob expansion into array
words=(one two "three four")     # quoted element preserved

# Assignment
fruits[3]="date"
fruits+=("elderberry")           # append

# Access
echo ${fruits[0]}                # apple
echo ${fruits[-1]}               # last element (Bash 4.3+)
echo ${fruits[@]}                # all elements
echo ${#fruits[@]}               # number of elements
echo ${!fruits[@]}               # all indexes

# Slicing
echo ${fruits[@]:1:2}            # banana cherry (from index 1, 2 elements)

# Iteration
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# Delete
unset 'fruits[1]'               # remove element at index 1 (sparse array!)
unset fruits                    # destroy entire array
```

### Associative Arrays (Bash 4+)

```bash
# MUST be explicitly declared
declare -A colors
colors[red]="#FF0000"
colors[green]="#00FF00"
colors[blue]="#0000FF"

# Or compound assignment
declare -A config=(
    [host]="localhost"
    [port]="8080"
    [debug]="true"
)

# Access
echo ${config[host]}             # localhost
echo ${config[@]}                # all values (unordered!)
echo ${!config[@]}               # all keys

# Check if key exists
if [[ -v config[debug] ]]; then
    echo "debug key exists"
fi

# Iteration over keys
for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done

# Delete
unset 'config[debug]'
```

### Array Operations

```bash
# Copy array
declare -a copy=("${original[@]}")

# Merge arrays
merged=("${arr1[@]}" "${arr2[@]}")

# Filter (Bash 4+)
filtered=()
for elem in "${arr[@]}"; do
    [[ $elem == *pattern* ]] && filtered+=("$elem")
done

# String to array (word splitting)
IFS=',' read -ra items <<< "one,two,three"

# Array to string
joined=$(IFS=','; echo "${arr[*]}")
```

## Arithmetic Expressions

### Syntax

```bash
# Arithmetic evaluation (command, returns exit code)
(( x = 5 + 3 ))
(( x++ ))
(( x > 10 )) && echo "big"

# Arithmetic expansion (substitution, produces value)
result=$(( 5 + 3 ))
echo "Result: $(( width * height ))"

# let builtin
let "x = 5 + 3"
let "x++" "y--"
```

### Operators (C-style, by precedence high→low)

| Category | Operators |
|----------|-----------|
| Post-increment/decrement | `id++` `id--` |
| Pre-increment/decrement | `++id` `--id` |
| Unary | `+` `-` |
| Logical/bitwise NOT | `!` `~` |
| Exponentiation | `**` |
| Multiplication/division | `*` `/` `%` |
| Addition/subtraction | `+` `-` |
| Bitwise shifts | `<<` `>>` |
| Comparison | `<` `>` `<=` `>=` |
| Equality | `==` `!=` |
| Bitwise AND/XOR/OR | `&` `^` `|` |
| Logical AND/OR | `&&` `||` |
| Ternary | `? :` |
| Assignment | `=` `+=` `-=` `*=` `/=` `%=` `<<=` `>>=` `&=` `^=` `|=` |
| Comma | `,` |

### Number Bases

```bash
echo $(( 0xFF ))        # 255 (hex)
echo $(( 077 ))         # 63  (octal, leading zero)
echo $(( 2#1010 ))      # 10  (binary)
echo $(( 8#77 ))        # 63  (octal, explicit base)
echo $(( 64#@_ ))       # max base is 64

# Force decimal interpretation
x=010
echo $(( 10#$x ))       # 10 (not 8)
```

### Truth in Arithmetic

```bash
# 0 = false, non-zero = true (opposite of exit codes!)
if (( x )); then echo "x is non-zero"; fi
if (( x > 5 )); then echo "x > 5"; fi

# Ternary operator
(( result = (x > 10) ? x : 10 ))
```

## Compound Commands

### Overview

| Category | Syntax | Description |
|----------|--------|-------------|
| **Grouping** | `{ ...; }` | Command grouping (current shell) |
| | `( ... )` | Command grouping in a subshell |
| **Conditionals** | `[[ ... ]]` | Conditional expression |
| | `if ...; then ...; fi` | Conditional branching |
| | `case ... esac` | Pattern-based branching |
| **Loops** | `for word in ...; do ...; done` | Classic for-loop |
| | `for ((x=1; x<=10; x++)); do ...; done` | C-style for-loop |
| | `while ...; do ...; done` | While loop |
| | `until ...; do ...; done` | Until loop |
| **Misc** | `(( ... ))` | Arithmetic evaluation |
| | `select word in ...; do ...; done` | User selections |

### Conditionals

```bash
# if/elif/else
if [[ condition ]]; then
    commands
elif [[ condition ]]; then
    commands
else
    commands
fi

# case (pattern matching)
case "$input" in
    start|begin)  do_start ;;
    stop|end)     do_stop ;;
    restart)      do_stop; do_start ;;
    *)            echo "Unknown: $input" ;;
esac
```

### Loops

```bash
# Classic for loop
for item in word1 word2 word3; do
    echo "$item"
done

# C-style for loop
for ((i = 0; i < 10; i++)); do
    echo "$i"
done

# while loop
while [[ condition ]]; do
    commands
done

# until loop (opposite of while)
until [[ condition ]]; do
    commands
done

# Read lines from file
while IFS= read -r line; do
    echo "$line"
done < file.txt

# Infinite loop
while true; do
    sleep 1
done
```

### Grouping

```bash
# Group in current shell (shares variables)
{ cmd1; cmd2; cmd3; }

# Group in subshell (isolated environment)
( cmd1; cmd2; cmd3 )

# Practical use: redirect a block
{
    echo "header"
    cat data.txt
    echo "footer"
} > output.txt
```

### Misc

```bash
# Arithmetic evaluation
(( x = 5 * 3 ))
(( x++ ))

# select - User menu
select choice in "Option1" "Option2" "Quit"; do
    case $choice in
        Option1) echo "You chose 1" ;;
        Option2) echo "You chose 2" ;;
        Quit)    break ;;
        *)       echo "Invalid" ;;
    esac
done
```

## Conditional Expressions

### [[ ... ]] (Bash Extended Test)

```bash
# String comparisons
[[ $str == "hello" ]]       # equality
[[ $str != "hello" ]]       # inequality
[[ $str == h* ]]            # glob pattern match
[[ $str =~ ^[0-9]+$ ]]     # regex match
[[ -z $str ]]               # true if empty
[[ -n $str ]]               # true if non-empty
[[ $a < $b ]]               # lexicographic less-than

# File tests
[[ -e $file ]]              # exists
[[ -f $file ]]              # regular file
[[ -d $path ]]              # directory
[[ -r $file ]]              # readable
[[ -w $file ]]              # writable
[[ -x $file ]]              # executable
[[ -s $file ]]              # non-empty (size > 0)
[[ -L $file ]]              # symlink
[[ $f1 -nt $f2 ]]          # f1 newer than f2
[[ $f1 -ot $f2 ]]          # f1 older than f2

# Logical operators
[[ cond1 && cond2 ]]        # AND
[[ cond1 || cond2 ]]        # OR
[[ ! condition ]]           # NOT
```

### (( ... )) (Arithmetic Test)

```bash
(( x > 5 ))
(( x >= 1 && x <= 100 ))
(( (x % 2) == 0 ))         # even number
```

## Redirection

| Syntax | Description |
|--------|-------------|
| `cmd > file` | Redirect stdout to file (overwrite) |
| `cmd >> file` | Redirect stdout to file (append) |
| `cmd 2> file` | Redirect stderr to file |
| `cmd 2>&1` | Redirect stderr to stdout |
| `cmd &> file` | Redirect both stdout and stderr to file |
| `cmd < file` | Redirect file to stdin |
| `cmd <<< "string"` | Here-string (feed string to stdin) |
| `cmd << EOF` | Here-document |
| `cmd1 \| cmd2` | Pipe stdout of cmd1 to stdin of cmd2 |
| `cmd1 \|& cmd2` | Pipe both stdout and stderr (Bash 4+) |

```bash
# Discard output
command > /dev/null 2>&1
command &> /dev/null         # shorter (Bash)

# Here-document
cat << 'EOF'
No $expansion here (quoted delimiter)
Everything is literal
EOF

cat << EOF
Variables are $expanded here
Command: $(whoami)
EOF

# Here-string
grep "pattern" <<< "$my_variable"

# Process substitution (treat command output as a file)
diff <(sort file1) <(sort file2)
while read -r line; do echo "$line"; done < <(command)
```

## Brace Expansion

```bash
# String lists
echo {a,b,c}          # a b c
echo file.{txt,log}   # file.txt file.log
mv file.{old,new}     # rename file.old to file.new

# Ranges
echo {1..10}          # 1 2 3 4 5 6 7 8 9 10
echo {a..z}           # a b c ... z
echo {01..10}         # 01 02 03 ... 10 (zero-padded)

# Ranges with step (Bash 4+)
echo {0..100..10}     # 0 10 20 30 ... 100
echo {a..z..2}        # a c e g i k m o q s u w y

# Combinations
echo {A,B}{1,2}       # A1 A2 B1 B2
mkdir -p project/{src,tests,docs}/{main,utils}
```

## Special Parameters

| Parameter | Description |
|-----------|-------------|
| `$0` | Script name / shell name |
| `$1` ... `$9` | Positional parameters (arguments) |
| `${10}` | 10th argument (braces required for > 9) |
| `$#` | Number of positional parameters |
| `$@` | All positional parameters (individually quoted) |
| `$*` | All positional parameters (as single word when quoted) |
| `$?` | Exit status of last command |
| `$$` | PID of current shell |
| `$!` | PID of last background command |
| `$-` | Current shell option flags |
| `$_` | Last argument of previous command |

```bash
# Difference between $@ and $*
set -- "arg one" "arg two" "arg three"

for arg in "$@"; do echo "$arg"; done
# arg one
# arg two
# arg three

for arg in "$*"; do echo "$arg"; done
# arg one arg two arg three  (single string!)
```

## Builtin Commands Reference

Builtins are provided directly by the shell, not invoked as external commands.

### Declaration Commands

Commands that set and query attributes/types, and manipulate simple datastructures.

| Command | Alt | Type | Description |
|---------|-----|------|-------------|
| `declare` | `typeset` | builtin | Display or set shell variables/functions along with attributes |
| `export` | `typeset -x` | special builtin | Display or set shell variables, giving them the export attribute |
| `eval` | - | special builtin | Evaluate arguments as shell code |
| `local` | - | builtin | Declare variables as having function local scope |
| `readonly` | `typeset -r` | special builtin | Mark variables or functions as read-only |
| `unset` | - | special builtin | Unset variables and functions |
| `shift` | - | special builtin | Shift positional parameters |

### I/O Commands

Commands for reading/parsing input, or producing/formatting output of standard streams.

| Command | Alt | Type | Description |
|---------|-----|------|-------------|
| `coproc` | - | keyword | Run a command in the background with pipes for reading/writing its streams |
| `echo` | - | builtin | Create output from arguments |
| `mapfile` | `readarray` | builtin | Read lines of input into an array |
| `printf` | - | builtin | Formatted output ("advanced echo") |
| `read` | - | builtin | Read input into variables or arrays, or split strings using delimiters |

### declare Flags Quick Reference

| Flag | Description |
|------|-------------|
| `-a` | Indexed array |
| `-A` | Associative array |
| `-i` | Integer attribute |
| `-l` | Lowercase on assignment |
| `-u` | Uppercase on assignment |
| `-r` | Readonly |
| `-x` | Export (same as `export`) |
| `-n` | Nameref (reference to another variable, Bash 4.3+) |
| `-p` | Display attributes and value of variable |

### Configuration and Debugging

Commands that modify shell behavior, change special options, assist in debugging.

| Command | Alt | Type | Description |
|---------|-----|------|-------------|
| `caller` | - | builtin | Identify/print execution frames |
| `set` | - | special builtin | Set positional parameters and/or options that affect shell behaviour |
| `shopt` | - | builtin | Set/get bash-specific shell options |

### Control Flow and Data Processing

Commands that operate on data and/or affect control flow.

| Command | Alt | Type | Description |
|---------|-----|------|-------------|
| `:` (colon) | `true` | special builtin | Null command ("true") |
| `.` (dot) | `source` | special builtin | Source external files |
| `false` | - | builtin | Fail at doing nothing (returns 1) |
| `continue` / `break` | - | special builtin | Continue with or break out of loops |
| `let` | - | builtin | Arithmetic evaluation simple command |
| `return` | - | special builtin | Return from a function with a specified exit status |
| `[` | `test` | builtin | The classic test simple command |

### Process and Job Control

Commands related to jobs, signals, process groups, subshells.

| Command | Alt | Type | Description |
|---------|-----|------|-------------|
| `exec` | - | special builtin | Replace the current shell process or set redirections |
| `exit` | - | special builtin | Exit the shell |
| `kill` | - | builtin | Send a signal to specified process(es) |
| `trap` | - | special builtin | Set signal handlers or output the current handlers |
| `times` | - | special builtin | Display process times |
| `wait` | - | builtin | Wait for background jobs and asynchronous lists |

### I/O Commands

```bash
# read - Read input
read -r line                     # read a line (raw, no backslash interpretation)
read -rp "Name: " name          # prompt and read
read -rs -p "Pass: " pass       # silent (password)
read -ra arr <<< "a b c"        # read into array
read -t 5 -r input              # timeout after 5 seconds
read -n 1 -r char               # read single character
read -d '' -r var < file        # read entire file

# printf - Formatted output
printf '%s\n' "hello"
printf '%d items at $%.2f\n' 5 3.14
printf -v result '%05d' 42      # store in variable (no subshell!)

# echo - Simple output
echo "hello world"
echo -n "no newline"            # suppress trailing newline
echo -e "tabs\there"            # interpret escape sequences
```

### Control Flow

```bash
# return - Exit from function
myfunc() { [[ $1 ]] || return 1; echo "$1"; }

# exit - Exit the script
exit 0    # success
exit 1    # failure

# break / continue
for i in {1..10}; do
    (( i == 5 )) && continue   # skip 5
    (( i == 8 )) && break      # stop at 8
    echo "$i"
done

# trap - Signal handling
trap 'rm -f "$tmpfile"' EXIT          # cleanup on exit
trap 'echo "Caught SIGINT"' INT       # handle Ctrl+C
trap '' INT                            # ignore SIGINT
trap - INT                             # reset to default
```

## Functions

```bash
# Declaration (two equivalent forms)
function greet {
    echo "Hello, $1"
}

greet() {
    echo "Hello, $1"
}

# With local variables
calculate() {
    local result=$(( $1 + $2 ))
    echo $result
}

# Return values (exit codes only: 0-255)
is_even() {
    (( $1 % 2 == 0 ))    # return code set by arithmetic
}
if is_even 4; then echo "even"; fi

# Capture output
sum=$(calculate 3 5)

# Pass array to function
process_array() {
    local -n arr_ref=$1    # nameref (Bash 4.3+)
    for item in "${arr_ref[@]}"; do
        echo "$item"
    done
}
my_array=(a b c)
process_array my_array
```

## Pattern Matching (Globs)

| Pattern | Matches |
|---------|---------|
| `*` | Any string (including empty) |
| `?` | Any single character |
| `[abc]` | Any one of a, b, or c |
| `[a-z]` | Any character in range |
| `[^abc]` or `[!abc]` | Any character NOT in set |

### Extended Globs (shopt -s extglob)

| Pattern | Matches |
|---------|---------|
| `?(pattern)` | Zero or one occurrence |
| `*(pattern)` | Zero or more occurrences |
| `+(pattern)` | One or more occurrences |
| `@(pattern)` | Exactly one occurrence |
| `!(pattern)` | Anything except the pattern |

```bash
shopt -s extglob

ls *.!(log)           # all files except *.log
rm !(important.txt)   # remove all except important.txt
echo +([0-9])         # one or more digits
```

### Glob Options

```bash
shopt -s nullglob     # unmatched globs expand to nothing (not literal)
shopt -s globstar     # ** matches directories recursively (Bash 4+)
shopt -s nocaseglob   # case-insensitive globbing
shopt -s dotglob      # include hidden files (.*) in globs

# Example with globstar
for file in **/*.py; do
    echo "$file"
done
```

## Common Pitfalls

### 1. Unquoted Variables

```bash
# WRONG - breaks on spaces and globs
for f in $files; do echo $f; done

# RIGHT - always quote
for f in "${files[@]}"; do echo "$f"; done
```

### 2. Word Splitting in Tests

```bash
# WRONG - fails if myvar has spaces
[ $myvar = test ]

# RIGHT - quote variables
[ "$myvar" = test ]

# BEST - use [[ ]] (no word splitting)
[[ $myvar == test ]]
```

### 3. Command Substitution Whitespace

```bash
# WRONG - trailing newlines stripped, word splitting happens
files=$(ls)

# RIGHT - use arrays and globs
files=(*)
```

### 4. Parsing ls Output

```bash
# WRONG - never parse ls
for f in $(ls *.txt); do echo "$f"; done

# RIGHT - use globs
for f in *.txt; do echo "$f"; done
```

### 5. Read Loop Losing Variables

```bash
# WRONG - pipe creates subshell, variables lost
cat file.txt | while read -r line; do count=$((count+1)); done
echo $count  # empty!

# RIGHT - redirect instead of pipe
while read -r line; do count=$((count+1)); done < file.txt
echo $count  # correct
```

### 6. Forgetting `--` for End of Options

```bash
# WRONG - file named "-rf" would be interpreted as options
rm $filename

# RIGHT - use -- to signal end of options
rm -- "$filename"
```

## Script Best Practices

```bash
#!/usr/bin/env bash

# Strict mode
set -euo pipefail

# -e  : exit on error
# -u  : error on undefined variables
# -o pipefail : pipe fails if any command fails

# Debug mode
set -x    # print every command before executing

# Cleanup trap
cleanup() { rm -f "$tmpfile"; }
trap cleanup EXIT

# Check dependencies
command -v jq &> /dev/null || { echo "jq required"; exit 1; }

# Safe temporary files
tmpfile=$(mktemp)

# Always quote variables
echo "${var:-default}"

# Use [[ ]] instead of [ ]
[[ -f "$file" ]] && source "$file"

# Use arrays for command arguments
args=(-v --color=always --exclude-dir=.git)
grep "${args[@]}" "pattern" .

# Prefer printf over echo
printf '%s\n' "$message"
```

## Useful One-Liners

```bash
# Check if variable is set
[[ -v varname ]] && echo "set"

# Check if command exists
command -v git &> /dev/null

# Default value with assignment
: "${TIMEOUT:=30}"

# Swap two variables
tmp=$a; a=$b; b=$tmp

# Trim whitespace
trimmed="${str#"${str%%[![:space:]]*}"}"

# Uppercase/lowercase (Bash 4+)
echo "${str^^}"   # UPPER
echo "${str,,}"   # lower

# Random number in range
echo $(( RANDOM % 100 + 1 ))   # 1-100

# Repeat a character
printf '=%.0s' {1..40}; echo   # 40 equal signs

# Check if running as root
(( EUID == 0 )) || { echo "Run as root"; exit 1; }

# Elapsed time
SECONDS=0
# ... do stuff ...
echo "Took $SECONDS seconds"
```

## Shell Options Reference

### set options

| Flag | Description |
|------|-------------|
| `set -e` | Exit on error |
| `set -u` | Error on unset variables |
| `set -x` | Print commands before execution (debug) |
| `set -o pipefail` | Pipeline fails if any command fails |
| `set -f` | Disable globbing |
| `set -n` | Syntax check only (no execution) |

### shopt options

| Option | Description |
|--------|-------------|
| `extglob` | Extended pattern matching |
| `globstar` | `**` recursive matching |
| `nullglob` | Unmatched globs expand to nothing |
| `nocaseglob` | Case-insensitive globs |
| `dotglob` | Globs match hidden files |
| `lastpipe` | Last pipe command runs in current shell |
| `checkwinsize` | Update LINES/COLUMNS after each command |
