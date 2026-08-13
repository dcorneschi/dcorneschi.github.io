# Korn Shell (ksh) Cheatsheet

The Korn Shell (ksh) is the default shell on many commercial Unix systems (AIX, HP-UX, Solaris). Two major versions exist: ksh88 (shipped with most classic Unix systems) and ksh93 (modern rewrite with extended features). Understanding ksh is essential for administering legacy Unix environments.

## Check ksh Version

```bash
# ksh93
ksh --version
echo ${.sh.version}
echo $KSH_VERSION

# ksh88 (AIX, Solaris) — no --version flag
# Use the emacs editing mode trick:
set -o emacs
# Then press Ctrl+V — it prints the version string
```

## History

### History File Defaults

| Setting | Root | Non-Root |
|---------|------|----------|
| Default history size | 500 lines | 128 lines |
| History file | `~/.sh_history` | `~/.sh_history` |
| Timestamp recording | Not recorded by default | Not recorded by default |

> **Note:** The history file does NOT record what date/time commands were run (unlike bash's `HISTTIMEFORMAT`). To enable timestamps in ksh93, export `EXTENDED_HISTORY=ON`.

### Configure History

```bash
# Set history file location
export HISTFILE=~/.sh_history

# Set history size
export HISTSIZE=1000

# Enable extended history with timestamps (ksh93)
export EXTENDED_HISTORY=ON
```

### History Commands (fc)

The `fc` (fix command) built-in is the primary way to interact with history in ksh:

```bash
# List all commands in history
fc -l

# List all commands with execution time (AIX ksh — requires EXTENDED_HISTORY=ON)
fc -lt

# List commands 10 to 20
fc -l 10 20

# List the last 10 commands
fc -l -10

# List all commands starting from command 200
fc -l 200

# List all commands since a specific command name
fc -l alias

# Edit a specific command with vi before re-running
fc -e vi 883

# Edit the last command with your $FCEDIT or $EDITOR
fc
```

### Re-Run Commands (r)

```bash
# Repeat the last command
r

# Repeat the last command starting with "topas"
r topas

# Repeat command number 14
r 14
```

### Search History

```bash
# Search history file for a pattern (handles binary/null bytes)
strings -n 1 ~/.sh_history | grep top

# Reverse search (emacs mode) — Ctrl+R
set -o emacs
# Then Ctrl+R and type search term

# Reverse search (vi mode) — ESC then / to search
set -o vi
# Press ESC, then /pattern, then Enter
```

## Editing Modes

ksh supports both vi and emacs command-line editing:

### Emacs Mode

```bash
# Enable emacs editing
set -o emacs

# Key bindings:
# Ctrl+A      — Move to beginning of line
# Ctrl+E      — Move to end of line
# Ctrl+F      — Move forward one character
# Ctrl+B      — Move backward one character
# Alt+F       — Move forward one word
# Alt+B       — Move backward one word
# Ctrl+D      — Delete character under cursor
# Ctrl+K      — Kill (cut) to end of line
# Ctrl+U      — Kill entire line
# Ctrl+W      — Kill word backward
# Ctrl+Y      — Yank (paste) killed text
# Ctrl+P      — Previous command (history up)
# Ctrl+N      — Next command (history down)
# Ctrl+R      — Reverse search history
# Ctrl+V      — Show ksh version (ksh88)
# Tab         — Command completion (ksh93 only)
```

### Vi Mode

```bash
# Enable vi editing
set -o vi

# In insert mode (default):
# ESC         — Switch to command mode
# Ctrl+V      — Show ksh version (ksh88)

# In command mode (after ESC):
# h / l       — Move left / right
# w / b       — Move word forward / backward
# 0 / $       — Move to beginning / end of line
# x           — Delete character
# dd          — Delete entire line
# dw          — Delete word
# cw          — Change word
# p           — Paste
# k / j       — Previous / next history command
# /pattern    — Search history backward
# ?pattern    — Search history forward
# n           — Repeat last search
# i / a       — Enter insert mode (before / after cursor)
# A           — Append at end of line
# I           — Insert at beginning of line
```

## Tab Completion

```bash
# ksh88 — NO tab completion available

# pdksh — enable vi-tabcomplete
set -o vi-tabcomplete

# ksh93 — enable viraw for vi mode tab completion
set -o viraw

# ksh93 in emacs mode — Tab works by default
set -o emacs
# Then Tab completes commands, files, and variables
```

| Shell | Tab Completion | How to Enable |
|-------|---------------|---------------|
| ksh88 | Not available | — |
| pdksh | Available | `set -o vi-tabcomplete` |
| ksh93 (vi mode) | Available | `set -o viraw` |
| ksh93 (emacs mode) | Available | Enabled by default |

## Variables

### Special Variables

| Variable | Description |
|----------|-------------|
| `$?` | Exit status of last command |
| `$$` | PID of current shell |
| `$!` | PID of last background process |
| `$#` | Number of positional parameters |
| `$@` | All positional parameters (individually quoted) |
| `$*` | All positional parameters (as one word) |
| `$0` | Script name |
| `$_` | Last argument of previous command |
| `$RANDOM` | Random number (ksh93) |
| `$SECONDS` | Seconds since shell started |
| `$LINENO` | Current line number in script |
| `$PWD` | Current working directory |
| `$OLDPWD` | Previous working directory |
| `$PPID` | Parent process ID |
| `$ENV` | File sourced at shell startup (interactive) |
| `${.sh.version}` | ksh93 version string |

### Variable Operations

```bash
# Default values
${var:-default}        # Use default if var is unset or null
${var:=default}        # Set var to default if unset or null
${var:+alternate}      # Use alternate if var IS set
${var:?error msg}      # Print error and exit if var unset

# String operations
${#var}                # Length of var
${var#pattern}         # Remove shortest prefix match
${var##pattern}        # Remove longest prefix match
${var%pattern}         # Remove shortest suffix match
${var%%pattern}        # Remove longest suffix match
${var/old/new}         # Replace first occurrence (ksh93)
${var//old/new}        # Replace all occurrences (ksh93)
${var:offset:length}   # Substring (ksh93)
```

## Arrays

```bash
# Indexed arrays
set -A colors red green blue
echo ${colors[0]}       # red
echo ${colors[1]}       # green
echo ${colors[@]}       # all elements
echo ${#colors[@]}      # number of elements

# ksh93 also supports:
typeset -a arr
arr=(one two three)
arr+=(four)             # Append

# Associative arrays (ksh93 only)
typeset -A capitals
capitals[France]=Paris
capitals[Germany]=Berlin
echo ${capitals[France]}
echo ${!capitals[@]}    # All keys
```

## Functions

```bash
# ksh88 style
function greet {
    print "Hello, $1"
}

# POSIX style (also works)
greet() {
    print "Hello, $1"
}

# Call
greet "World"

# Local variables (ksh93)
function myfunc {
    typeset local_var="local"
    print $local_var
}

# Return value
function add {
    typeset -i result=$1+$2
    print $result
}
total=$(add 3 5)
```

## print vs echo

ksh uses `print` as its primary output command (more portable than `echo` across Unix systems):

```bash
# print (ksh built-in — preferred)
print "Hello World"
print -n "No newline"       # No trailing newline
print -r "Raw: \n no escape"  # Don't interpret escape sequences
print -- "-flag-like-text"  # Print text starting with -

# printf (formatted output)
printf "%s is %d years old\n" "Alice" 30
printf "%-20s %s\n" "Name" "Value"
```

## Control Flow

### if/elif/else

```bash
if [[ $x -gt 10 ]]; then
    print "Greater than 10"
elif [[ $x -eq 10 ]]; then
    print "Equal to 10"
else
    print "Less than 10"
fi
```

### case

```bash
case $answer in
    yes|y|Y)  print "Agreed" ;;
    no|n|N)   print "Declined" ;;
    *)        print "Unknown" ;;
esac
```

### Loops

```bash
# for loop
for f in *.txt; do
    print "Processing: $f"
done

# C-style for (ksh93)
for ((i=0; i<10; i++)); do
    print $i
done

# while
while read line; do
    print "$line"
done < file.txt

# until
until [[ $count -ge 5 ]]; do
    ((count++))
done

# select (interactive menu)
select choice in "Option 1" "Option 2" "Quit"; do
    case $REPLY in
        1) print "Selected Option 1" ;;
        2) print "Selected Option 2" ;;
        3) break ;;
    esac
done
```

## Arithmetic

```bash
# Integer arithmetic with (( ))
typeset -i x=5
((x = x + 3))
((x++))
((x += 10))

# let command
let x=5+3
let "x = x * 2"

# Arithmetic substitution
result=$((5 + 3 * 2))
print $((10 / 3))        # Integer division
print $((10 % 3))        # Modulo
```

## Test Conditions

```bash
# [[ ]] — preferred in ksh (extended test)
[[ -f /etc/passwd ]]     # File exists and is regular
[[ -d /tmp ]]            # Directory exists
[[ -r file ]]            # File is readable
[[ -w file ]]            # File is writable
[[ -x file ]]            # File is executable
[[ -s file ]]            # File exists and is not empty
[[ -L file ]]            # Symbolic link
[[ -z "$var" ]]          # String is empty
[[ -n "$var" ]]          # String is not empty
[[ "$a" == "$b" ]]       # String equality
[[ "$a" != "$b" ]]       # String inequality
[[ "$a" == pattern* ]]   # Pattern matching (ksh93)
[[ "$a" =~ regex ]]      # Regex matching (ksh93)
[[ $x -eq $y ]]          # Numeric equality
[[ $x -lt $y ]]          # Numeric less than
[[ $x -gt $y ]]          # Numeric greater than
```

## I/O Redirection

```bash
# Standard redirections
command > file            # Stdout to file (overwrite)
command >> file           # Stdout to file (append)
command 2> file           # Stderr to file
command 2>&1              # Stderr to stdout
command > file 2>&1       # Both stdout and stderr to file
command < file            # Stdin from file

# Here document
cat <<EOF
Hello $USER
Today is $(date)
EOF

# Here string (ksh93)
grep pattern <<< "$variable"

# Coprocess (ksh specific — run command in background with I/O pipes)
command |&               # Start coprocess
print -p "input"         # Write to coprocess stdin
read -p output           # Read from coprocess stdout
```

## Job Control

```bash
# Background a command
command &

# List jobs
jobs
jobs -l                  # With PIDs

# Bring to foreground
fg %1
fg %command_name

# Send to background
bg %1

# Wait for background jobs
wait                     # Wait for all
wait $pid                # Wait for specific PID
wait %1                  # Wait for job number
```

## Startup Files

| File | When Sourced |
|------|-------------|
| `/etc/profile` | Login shells (all users) |
| `~/.profile` | Login shells (user-specific) |
| `$ENV` file | Interactive shells (set ENV=~/.kshrc in .profile) |

```bash
# Typical .profile setup
export ENV=~/.kshrc
export HISTFILE=~/.sh_history
export HISTSIZE=1000
export EDITOR=vi
export VISUAL=vi
set -o vi
```

```bash
# Typical .kshrc (sourced for interactive shells)
alias ll='ls -la'
alias la='ls -A'
alias h='fc -l'

PS1='${USER}@${HOSTNAME}:${PWD##*/} $ '
```

## ksh88 vs ksh93 Differences

| Feature | ksh88 | ksh93 |
|---------|-------|-------|
| Tab completion | No | Yes |
| `${var/old/new}` | No | Yes |
| `${var:offset:length}` | No | Yes |
| Associative arrays | No | Yes |
| `$RANDOM` | No | Yes |
| Regex in `[[ ]]` | No | `=~` supported |
| `$'...'` quoting | No | Yes |
| Floating point | No | Yes (`typeset -F`) |
| Compound variables | No | Yes |
| C-style `for` loop | No | Yes |
| Here strings `<<<` | No | Yes |
| Coprocess | `|&` | `|&` (enhanced) |
| `--version` flag | No | Yes |
| Nameref variables | No | `typeset -n` |

## Differences from bash

| Feature | ksh | bash |
|---------|-----|------|
| History command | `fc -l`, `r` | `history`, `!` |
| Repeat last | `r` | `!!` |
| Repeat by number | `r 14` | `!14` |
| Repeat by name | `r topas` | `!topas` |
| Print command | `print` (built-in) | `echo` |
| History timestamps | `EXTENDED_HISTORY=ON` | `HISTTIMEFORMAT` |
| Default history file | `~/.sh_history` | `~/.bash_history` |
| Startup (interactive) | `$ENV` file | `~/.bashrc` |
| Coprocess | `cmd |&` | `coproc cmd` |
| Array assign | `set -A arr val1 val2` | `arr=(val1 val2)` |

## One-Liners

```bash
# Show history with line numbers
fc -l

# Re-run last command
r

# Re-run last ssh command
r ssh

# Search history for a pattern
strings -n 1 ~/.sh_history | grep "pattern"

# Show ksh version (works in ksh93)
echo ${.sh.version}

# Set vi mode permanently
echo 'set -o vi' >> ~/.profile

# Enable extended history with timestamps
echo 'export EXTENDED_HISTORY=ON' >> ~/.profile

# Quick function to search history
function hg { fc -l 1 | grep "$1"; }

# Reload kshrc
. ~/.kshrc

# Check if running ksh
echo $0                    # Shows -ksh or ksh
ps -p $$ -o comm=          # Shows the actual shell binary
```

## Useful Settings

```bash
# Put in ~/.profile or ~/.kshrc

set -o vi                    # Vi editing mode
set -o emacs                 # Emacs editing mode (choose one)
set -o ignoreeof             # Prevent Ctrl+D from exiting
set -o noclobber             # Prevent > from overwriting files (use >| to force)
set -o trackall              # Track all aliases for hash table
set -o markdirs              # Append / to directories in completion

# Script safety
set -o errexit               # Exit on error (set -e)
set -o nounset               # Error on undefined variables (set -u)
set -o pipefail              # Pipeline fails if any command fails (ksh93)
```

## Quick Reference

```bash
# History
fc -l                        # List history
fc -l -10                    # Last 10 commands
fc -l 10 20                  # Commands 10 to 20
fc -e vi 883                 # Edit command 883 in vi
r                            # Repeat last command
r topas                      # Repeat last 'topas' command
r 14                         # Repeat command 14

# Editing mode
set -o vi                    # Vi mode
set -o emacs                 # Emacs mode

# Variables
typeset -i num=5             # Integer
typeset -l lower="ABC"       # Lowercase
typeset -u upper="abc"       # Uppercase
typeset -r const="fixed"     # Read-only
typeset -x exported="val"    # Export
typeset -F float=3.14        # Float (ksh93)

# Version
ksh --version                # ksh93
echo ${.sh.version}          # ksh93
echo $KSH_VERSION            # pdksh/mksh
```
