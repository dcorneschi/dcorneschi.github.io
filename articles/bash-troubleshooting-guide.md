# Bash Troubleshooting Guide

<img src="/articles/images/bash-logo.svg" alt="Bash" width="150">

## 1. Enable bash debug mode with set -x

This shows each command before it's executed:
```bash
set -x          # Enable debug mode
# your commands here
set +x          # Disable debug mode
```

## 2. Run bash scripts with debug flag

```bash
bash -x script.sh
```

## 3. Add debug shebang to scripts

Add this at the top of your bash script:
```bash
#!/bin/bash -x
```

## 4. Verbose mode with set -v

This prints shell input lines as they are read:
```bash
set -v          # Enable verbose mode
set +v          # Disable verbose mode
```

## 5. Combine multiple debug options

```bash
set -xv         # Both debug and verbose
# or
bash -xv script.sh
```

## 6. For more detailed debugging, use set -euxo pipefail

- `-e`: Exit on error
- `-u`: Exit on undefined variable
- `-x`: Print commands before executing
- `-o pipefail`: Fail on pipe errors

```bash
set -euxo pipefail
```

## Common Usage Examples

### Debugging an entire script

```bash
#!/bin/bash
set -x

echo "Starting script..."
cd /some/directory
ls -la
echo "Script completed"
```

### Syntax check with set -n

Parse a script without executing it — catches missing `fi`, `done`, unclosed quotes:
```bash
bash -n script.sh
```

### Redirect debug output to a file

```bash
bash -x script.sh 2>debug.log
```

### Debug in background with log

```bash
# Trace goes to debug.log, stdout still prints to terminal
(set -x; ./script.sh) 2>debug.log &

# Both stdout and trace go to file (completely silent)
(set -x; ./script.sh) >debug.log 2>&1 &
```

### Inspect a function definition

```bash
declare -f my_function  # Show full function body
declare -F              # List all defined function names
```

## Notes

- The most commonly used option is `set -x` which shows commands with variable expansions
- Use `set +x` to disable debugging mode
- Debug output goes to stderr, so you can redirect it: `bash -x script.sh 2>debug.log`
- The `$-` variable contains current shell options (e.g. `himBHs` means `-h -i -m -B -H -s` are active)
- For production scripts, consider using conditional debugging with environment variables
- Use subshells `(set -x; ./script.sh)` to isolate debug settings from the parent shell

## Best Practices

1. **Most common approach**: `set -x` at start, `set +x` at end
2. **For production code**: Use environment variable toggle (`DEBUG=1`)
3. **For isolated tracing**: Use subshells `(set -x; ./script.sh)`
4. **For complex debugging**: Customize `PS4` variable

## Debugging Functions

### 1. Enable debug mode before calling the function

Define or source the function first, then wrap the call with `set -x` / `set +x`:
```bash
# Define the function (or source it from a file)
my_function() {
    echo "doing work"
    ls -la /tmp
}

# Trace only this call — everything else runs silently
set -x; my_function; set +x
```

To extract and trace a single function from a script that has a main body:
```bash
bash -x -c "$(bash -c 'source ./debug_demo.sh >/dev/null 2>&1; declare -f cleanup'); cleanup"
```
> Pass arguments after the function call if needed. Note: if the function depends on global variables defined in the script (like `$APP` or `$WORKDIR`), they won't be available — pass them explicitly or hardcode the values.

### 2. Use a subshell with debug mode
```bash
(set -x; my_function)
```
> The function must already be defined in the current shell. Check with `declare -f my_function`. The subshell inherits it and isolates `set -x` so it doesn't leak into your session.

### 3. Use environment variables

```bash
DEBUG=1 my_function
# or
DEBUG_FUNCTION=1 my_function
```
> This only works if the function internally checks `$DEBUG` or `$DEBUG_FUNCTION` and enables `set -x` based on it. Just setting the variable does nothing by itself.

Example implementation inside a script:
```bash
#!/bin/bash
# Enable debugging if DEBUG environment variable is set
if [[ "${DEBUG:-}" == "1" ]]; then
    set -x
fi

# Your script commands here
```

### 4. Debug multiple function calls

```bash
set -x
my_function1
my_function2 arg1
my_function3
set +x
```

## PS4 Customization

Customize the debug prompt to show file name and line number in trace output:

```bash
export PS4='+${BASH_SOURCE}:${LINENO}: '
set -x
```

---

## Sample Script to Test These Commands

Save this as `debug_demo.sh` and run it with different options to see each feature in action:

```bash
#!/bin/bash

APP="myapp"
WORKDIR="/tmp/${APP}_build"

create_directory() {
    local dir="$1"
    echo "Creating directory: $dir"
    mkdir -p "$dir"
}

deploy_files() {
    local target="$1"
    echo "Deploying $APP to $target"
    echo "version=1.0" > "$target/config.txt"
    cp "$target/config.txt" "$target/config.bak"
    ls -la "$target"
}

validate() {
    local target="$1"
    if [[ -f "$target/config.txt" ]]; then
        echo "Build OK"
    else
        echo "Build FAILED" >&2
        return 1
    fi
}

cleanup() {
    echo "Cleaning up $WORKDIR"
    rm -rf "$WORKDIR"
}

echo "=== Building $APP ==="
create_directory "$WORKDIR"
deploy_files "$WORKDIR"
validate "$WORKDIR"
cleanup
echo "=== Done ==="
```

### How to test each debugging method

```bash
# 1. Trace all commands (set -x)
bash -x debug_demo.sh

# 2. Verbose mode — see raw lines before expansion
bash -v debug_demo.sh

# 3. Combined trace + verbose
bash -xv debug_demo.sh

# 4. Syntax check only (set -n) — no execution
bash -n debug_demo.sh

# 5. Trace output to a file (keeps terminal clean)
bash -x debug_demo.sh 2>trace.log && cat trace.log

# 6. Custom PS4 with function name and line numbers
PS4='+${BASH_SOURCE}:${LINENO}: ' bash -x debug_demo.sh

# 7. Strict mode — catch errors, undefined vars, pipe failures
bash -euo pipefail debug_demo.sh
```
