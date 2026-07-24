# Bash Subshells

<img src="/articles/images/bash-logo.svg" alt="Bash" width="150">

A subshell is a child process spawned by the current shell. It inherits a copy of the parent's environment (variables, functions, file descriptors) but runs in its own isolated process. Changes made inside a subshell do not affect the parent shell.


## How Subshells Are Created

### Explicit Subshell with `( )`

Wrapping commands in parentheses runs them in a subshell:

```bash
# Changes inside ( ) don't affect the parent
cd /tmp
(cd /var; echo "Inside: $PWD")   # Inside: /var
echo "Outside: $PWD"             # Outside: /tmp
```

### Command Substitution `$( )`

Each command substitution runs in a subshell:

```bash
result=$(echo "hello")
# The echo ran in a subshell; $result is set in the parent
```

### Pipelines

Each command in a pipeline runs in its own subshell:

```bash
# 'read' runs in a subshell — the variable is lost
echo "hello" | read var
echo "$var"    # Empty — var was set in the subshell, not the parent
```

> In bash 4.2+ with `shopt -s lastpipe`, the last command in a pipeline runs in the current shell (not a subshell) when job control is disabled.

### Background Processes `&`

Commands run in the background execute in a subshell:

```bash
(sleep 10; echo "done") &
```

### Process Substitution `<( )` and `>( )`

```bash
diff <(ls /dir1) <(ls /dir2)
# Each ls runs in its own subshell
```

### Co-Processes `coproc`

A co-process creates a subshell with a two-way pipe, allowing the parent to send input and read output from the child process:

```bash
# Start a co-process
coproc MY_PROC { bash; }

# Send a command to the co-process
echo "echo hello from coproc" >&${MY_PROC[1]}

# Read the output
read -r output <&${MY_PROC[0]}
echo "$output"   # hello from coproc
```

### Replacing the Shell with `exec`

`exec` replaces the current shell process entirely — no subshell is spawned and no new nesting level is added:

```bash
# This replaces the current shell with a new bash instance
exec bash

# Useful for re-reading config without nesting
exec bash --login
```

> After `exec`, the previous shell no longer exists. There is nothing to `exit` back to.

### Visualizing Shell Nesting with `ps --forest`

Use `ps --forest` to see the parent/child hierarchy of shell processes:

```bash
bash
bash
ps --forest
#  PID TTY      STAT   TIME COMMAND
# 1234 pts/0    Ss     0:00 -bash
# 1235 pts/0    S      0:00  \_ bash
# 1236 pts/0    S      0:00      \_ bash
# 1237 pts/0    R+     0:00          \_ ps --forest
```


## Subshell vs Current Shell

| Feature | Subshell `( )` | Current shell `{ }` |
|---------|:--------------:|:-------------------:|
| Spawns new process | yes | no |
| Inherits variables | yes (copy) | yes (same) |
| Changes affect parent | no | yes |
| Inherits functions | yes | yes |
| Has its own PID | yes (`$BASHPID` differs) | no (`$BASHPID` same) |
| Requires semicolons/newlines | no | yes (and trailing `;`) |

```bash
# Subshell — changes don't leak
x=1
(x=99; echo "inside: $x")   # inside: 99
echo "outside: $x"           # outside: 1

# Group command — changes persist
x=1
{ x=99; echo "inside: $x"; }  # inside: 99
echo "outside: $x"             # outside: 99
```


## Detecting a Subshell

```bash
# $$ always returns the parent shell's PID
# $BASHPID returns the current process PID
echo "Parent PID: $$"
echo "Current PID: $BASHPID"

(echo "Subshell PID: $BASHPID, Parent: $$")
# Subshell PID will differ from Parent PID
```

You can also check `$BASH_SUBSHELL` which tracks the nesting level:

```bash
echo "Level: $BASH_SUBSHELL"       # 0
(echo "Level: $BASH_SUBSHELL")     # 1
((echo "Level: $BASH_SUBSHELL"))   # 2
```


## The Pipeline Variable Problem

The most common subshell pitfall — variables set inside a pipeline are lost:

```bash
# WRONG — count stays 0 because the while loop runs in a subshell
count=0
cat file.txt | while read line; do
    ((count++))
done
echo "$count"   # 0
```

### Solutions

**1. Use process substitution (avoids the pipe):**

```bash
count=0
while read line; do
    ((count++))
done < <(cat file.txt)
echo "$count"   # Correct count
```

**2. Use a here-string or redirect:**

```bash
count=0
while read line; do
    ((count++))
done < file.txt
echo "$count"   # Correct count
```

**3. Use `lastpipe` (bash 4.2+):**

```bash
shopt -s lastpipe
set +m   # Disable job control (required for lastpipe)

count=0
cat file.txt | while read line; do
    ((count++))
done
echo "$count"   # Correct count
```


## Practical Uses

### Isolate Side Effects

Run commands without affecting the current shell's state:

```bash
# Temporarily change directory and environment
(
    cd /opt/app
    export PATH="/opt/app/bin:$PATH"
    ./deploy.sh
)
# Still in original directory with original PATH
```

### Isolate Debug Output

```bash
(set -x; some_complex_function)
# Tracing is limited to the subshell — doesn't leak into your session
```

### Parallel Execution

```bash
(task1) &
(task2) &
(task3) &
wait    # Wait for all background subshells to finish
echo "All tasks completed"
```

### Temporary Environment Variables

```bash
# Set a variable for a single command without exporting
(MY_VAR=test ./my_script.sh)

# Or simply (no subshell needed for this case):
MY_VAR=test ./my_script.sh
```

### Atomic Operations

Group commands so they either all succeed or fail together:

```bash
(
    set -e
    cp config.bak config.new
    sed -i 's/old/new/g' config.new
    mv config.new config
) || echo "Update failed — original config unchanged"
```


## Performance Considerations

Subshells involve forking a new process, which has overhead. Avoid unnecessary subshells in tight loops:

```bash
# SLOW — spawns a subshell for each iteration
for file in *.txt; do
    size=$(wc -l < "$file")   # subshell per iteration
    echo "$file: $size"
done

# FASTER — single subshell for all files
wc -l *.txt
```

Common unnecessary subshell patterns:

```bash
# Unnecessary — echo in subshell
result=$(echo "$var" | tr 'a-z' 'A-Z')

# Better — use bash built-in
result="${var^^}"

# Unnecessary — cat in subshell (useless use of cat)
result=$(cat file.txt | grep "pattern")

# Better — grep reads the file directly
result=$(grep "pattern" file.txt)
```


## Summary

| Construct | Creates subshell | Use case |
|-----------|:----------------:|----------|
| `( commands )` | yes | Isolate side effects |
| `$( command )` | yes | Capture command output |
| `cmd1 \| cmd2` | yes (each) | Pipeline processing |
| `command &` | yes | Background execution |
| `<( command )` | yes | Process substitution |
| `{ commands; }` | no | Group without isolation |
| `source script` | no | Run in current shell |
| `. script` | no | Same as source |


## References

- [GNU Bash Manual — Command Grouping](https://www.gnu.org/software/bash/manual/bash.html#Command-Grouping)
- [BashFAQ — I set variables in a pipeline. Why do they disappear?](https://mywiki.wooledge.org/BashFAQ/024)
- [Bash Hackers Wiki — Process Substitution](https://bash-hackers.gabe565.com/syntax/expansion/proc_subst/)
