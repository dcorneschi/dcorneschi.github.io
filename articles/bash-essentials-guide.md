# Bash Essentials Guide

<img src="/articles/images/bash-logo.svg" alt="Bash" width="150">

A comprehensive reference covering shell sessions, environment variables, quoting, debugging, customization, and keyboard shortcuts.


## Shell Sessions

Bash sessions are classified along two independent axes: **login vs non-login** and **interactive vs non-interactive**. This gives four possible combinations, each with different startup behavior.

### Configuration Files

#### Login shell startup files

| Order | File | Scope |
|-------|------|-------|
| 1 | `/etc/profile` | System-wide |
| 2 | `~/.bash_profile` | User-specific (first found wins) |
| 3 | `~/.bash_login` | Fallback if `~/.bash_profile` doesn't exist |
| 4 | `~/.profile` | Fallback if neither of the above exist |

Only the **first** user-specific file found is executed. Most default `~/.bash_profile` scripts source `~/.bashrc` as well.

> **Note:** Ubuntu/Debian systems don't ship with `~/.bash_profile` by default. They use `~/.profile` instead, which is shell-agnostic (works with sh, dash, bash) and sources `~/.bashrc` if running bash. RHEL/CentOS/Fedora systems typically use `~/.bash_profile`.

#### Non-login shell startup files

| Order | File | Scope |
|-------|------|-------|
| 1 | `/etc/bash.bashrc` | System-wide |
| 2 | `~/.bashrc` | User-specific |

#### Non-interactive shell startup files

Non-interactive shells do **not** read any of the above files. They only read the file pointed to by the `BASH_ENV` environment variable, if set.


### 1. Interactive Login Shell

The shell you get when you first log into a system. It provides a prompt and reads login startup files.

```bash
ssh user@host                   # SSH into a remote machine
sudo su -                       # Switch to root with a full login environment
su - username                   # Switch user with login environment
bash --login                    # Explicitly start a login shell
login                           # The login program itself
```

Test it:

```bash
bash --login -i -c 'echo "$- | $(shopt login_shell)"'
```

### 2. Interactive Non-Login Shell

The shell you get when you open a new terminal window or start a subshell within an existing session. It provides a prompt but only reads `~/.bashrc`.

```bash
bash                            # Start a new bash subprocess
sudo su                         # Switch to root without login
tmux / screen                   # Shells inside terminal multiplexers
xterm / gnome-terminal          # New terminal window in a GUI
docker exec -it container bash  # Interactive shell inside a container
```

Test it:

```bash
bash -i -c 'echo "$- | $(shopt login_shell)"'
```

### 3. Non-Interactive Login Shell

A rare combination — a login shell that runs commands without a prompt. Can occur when piping commands to SSH.

```bash
echo 'echo $HOME' | ssh user@host       # Piping stdin to SSH
ssh user@host 'echo $HOME'              # Remote command via SSH (some configs)
bash --login -c 'echo hello'            # Explicitly: login + command string
```

Test it:

```bash
bash --login -c 'echo "$- | $(shopt login_shell)"'
```

### 4. Non-Interactive Non-Login Shell

The most common non-interactive case — running scripts, cron jobs, or command substitutions. No login files, no `~/.bashrc`, no prompt.

```bash
bash script.sh                          # Running a script file
bash -c 'echo hello'                    # Execute a string as a command
su user -c /path/to/command             # Run a single command as another user
sudo bash -c 'echo 0 > /proc/sys/kernel/yama/ptrace_scope'
./script.sh                             # Script with #!/bin/bash shebang
$(command)                              # Command substitution
cron jobs                               # Scripts executed by the cron daemon
find . -exec bash -c '...' \;          # Subshells spawned by other commands
```

Test it:

```bash
bash -c 'echo "$- | $(shopt login_shell)"'
```


### Detecting Your Shell Type

Use these commands to determine what kind of shell you're in:

```bash
# Check if the shell is interactive (look for 'i' in the output)
echo $-
# himBHs → interactive | hBs → non-interactive

# Check if it's a login shell
shopt login_shell
# login_shell    on  → login shell
# login_shell    off → non-login shell

# Alternative: login shells prepend '-' to $0
echo $0
# -bash → login shell | bash → non-login shell
```

#### Summary

| Shell Type | `echo $-` contains `i` | `shopt login_shell` | `echo $0` |
|------------|:----------------------:|:-------------------:|:---------:|
| Interactive login | yes | on | `-bash` |
| Interactive non-login | yes | off | `bash` |
| Non-interactive login | no | on | `bash` |
| Non-interactive non-login | no | off | `bash` |

#### Understanding `$-` Flags

The `$-` variable contains single-letter flags representing active shell options. Here's what each means:

| Flag | Name | Description |
|------|------|-------------|
| `h` | hashall | Cache command locations found via PATH |
| `i` | interactive | Shell accepts user input (has a prompt) |
| `m` | monitor | Job control enabled (bg, fg, CTRL+Z) |
| `B` | braceexpand | Brace expansion enabled (`{a,b,c}`) |
| `H` | histexpand | History expansion with `!` enabled |
| `c` | command | Shell was invoked with `-c` flag |
| `s` | stdin | Shell reads commands from stdin |

Interactive shells (`himBHc`) include `i`, `m`, and `H` for user interaction, job control, and history. Non-interactive shells (`hBc`) strip these since there's no user at the prompt.


## Standard File Descriptors

Every time a shell starts, three files are automatically opened:

| FD | Name   | Description     |
|----|--------|-----------------|
| 0  | stdin  | Standard input  |
| 1  | stdout | Standard output |
| 2  | stderr | Standard error  |

Display them for the current shell:

```bash
lsof +f g -ap $BASHPID -d 0,1,2
```


## Environment Variables

### Shell Variables vs Environment Variables

**Shell variables** are local to the current shell process and not passed to child processes. **Environment variables** are exported and inherited by all child processes.

```bash
MY_VAR="hello"          # Shell variable — only exists in this shell
bash -c 'echo $MY_VAR'  # Empty — child process can't see it

export MY_VAR="hello"   # Environment variable — passed to children
bash -c 'echo $MY_VAR'  # Prints "hello"
```

| | Shell variable | Environment variable |
|---|---|---|
| Created with | `VAR=value` | `export VAR=value` |
| Visible to child processes | no | yes |
| Listed by `env` / `printenv` | no | yes |
| Listed by `set` | yes | yes (both) |
| Promote to environment | — | `export VAR` |
| Demote to shell variable | — | `export -n VAR` |
| Remove completely | `unset VAR` | `unset VAR` |

### Viewing Variables

| Command | Description |
|---------|-------------|
| `printenv` or `env` | Display all currently set environment variables |
| `printenv TERM` or `echo $TERM` | Display a specific environment variable |
| `printenv SHELL USER` | Print multiple variables |
| `set \| less -N` | Display shell variables as well as environment variables |
| `echo $BASHOPTS` | List of options included when bash was compiled |
| `echo $SHELLOPTS` | Shell options set with the `set` command |

### Managing Variables

```bash
# Create a shell variable (local to current session)
EXAMPLE_VAR='hello world!'

# Promote it to an environment variable
export EXAMPLE_VAR

# Create and export in one step
export EXAMPLE_VAR='hello world!'

# Demote back to a shell variable
export -n EXAMPLE_VAR

# Remove completely
unset EXAMPLE_VAR
```


## Quoting and Escaping

Bash provides three quoting mechanisms: the escape character (`\`), single quotes, and double quotes.

**Single quotes** preserve the literal value of every character within them. A single quote cannot appear inside single quotes.

**Double quotes** preserve the literal value of all characters except `$`, `` ` ``, and `\`.

### Characters That Need Escaping

```
Whitespace (space, tab, newline)
! — history expansion
" — shell syntax
# — comment start when preceded by whitespace; zsh wildcards
$ — shell syntax
& — shell syntax
' — shell syntax
( — ksh/bash/zsh extended globs
) — closes (
* — sh wildcard
, — only inside brace expansion
; — shell syntax
< — shell syntax
= — in zsh, filename expansion with PATH lookup at beginning of filename
> — shell syntax
? — sh wildcard
[ — sh wildcard
\ — shell syntax
] — usually safe unquoted
^ — history expansion; zsh wildcard
` — shell syntax
{ — brace expansion
| — shell syntax
} — needs escaping in zsh
~ — home directory expansion at beginning of filename; zsh wildcard
```

### Characters Requiring Special Handling

| Character | Notes |
|-----------|-------|
| `-` | Not special to the shell, but indicates an option at the start of a command argument. Prefix with `./` to protect filenames starting with `-`. |
| `.` | Not special itself, but dot files are excluded from `*` globs by default. |
| `:` | Not special to the shell, but some commands parse it specially (e.g., `hostname:filename`). |


## I/O Redirection

```bash
# Redirect stderr and stdout to a file
command &> file
command > file 2>&1

# Redirect stderr and stdout to separate files
command > file1 2> file2

# Redirect stderr and stdout to /dev/null
command > /dev/null 2>&1
command &> /dev/null
```

### Piping Both stdout and stderr

Standard pipes (`|`) only capture stdout. If a command writes to stderr, it skips the pipe and prints directly to the terminal, missing your log file. Use `|&` to pipe **both** stdout and stderr (shorthand for `2>&1 |`):

```bash
# Only stdout goes to tee — stderr still prints to terminal
command | tee file.log

# Both stdout and stderr go to tee — nothing is lost
command |& tee file.log

# Equivalent longhand
command 2>&1 | tee file.log
```


## Debugging

### Running Scripts in Debug Mode

```bash
bash -x test.sh 2>&1 | tee out.test   # Trace execution
bash -n test.sh                         # Syntax check without execution
```

### Debug Options

| Command | Description |
|---------|-------------|
| `set -x` | Print statements after interpreting metacharacters and variables |
| `set +x` | Stop printing statements |
| `set -v` | Print statements before interpretation |
| `set -f` | Disable filename generation (globbing) |

### Debug a Portion of a Script

```bash
set -x
echo "Your home is: $HOME"
set +x
```

The `set -xv` combination shows statements both before and after interpretation, letting you see how the shell transforms each line.


## Shell Options and Flags

### Display Current Flags

```bash
echo $-
set -o
```

You can modify flags with `set`: `-` turns a flag **on**, `+` turns it **off**.

### Default Flags

| Flag | Name | Description |
|------|------|-------------|
| `h` | hashall | Remember locations of commands found via PATH |
| `i` | interactive | Shell is interactive |
| `m` | monitor | Enables job control (bg, fg) |
| `B` | braceexpand | Enables brace expansion |
| `H` | histexpand | Enables history expansion with `!` |

### Shell Options (shopt)

```bash
shopt           # Display all shell options
shopt -s        # Display enabled options
shopt -u        # Display disabled options
```

#### Commonly Used shopt Options

| Option | Description |
|--------|-------------|
| `cdspell` | Autocorrect minor typos in `cd` directory names |
| `checkwinsize` | Update LINES and COLUMNS after each command to match terminal size |
| `cmdhist` | Save multi-line commands as a single history entry |
| `dotglob` | Include filenames starting with `.` in glob expansion |
| `extglob` | Enable extended pattern matching (`?(pat)`, `*(pat)`, `+(pat)`, `@(pat)`, `!(pat)`) |
| `globstar` | Enable `**` to match all files and directories recursively |
| `histappend` | Append to history file instead of overwriting on shell exit |
| `nocaseglob` | Case-insensitive pathname expansion |
| `nocasematch` | Case-insensitive matching in `case` and `[[` commands |
| `nullglob` | Patterns that match no files expand to nothing instead of themselves |
| `failglob` | Patterns that match no files produce an error instead of expanding literally |
| `autocd` | Typing a directory name alone acts as `cd dirname` |
| `dirspell` | Autocorrect directory names during word completion |
| `extdebug` | Enable extended debugging (shows source file/line in `declare -F`) |
| `lithist` | Save multi-line commands with embedded newlines rather than semicolons |

#### Enable/Disable Examples

```bash
shopt -s cdspell        # Enable: auto-correct cd typos
shopt -u cdspell        # Disable it

shopt -s globstar       # Enable: recursive ** globbing
shopt -s extglob        # Enable: extended pattern matching
shopt -s nocaseglob     # Enable: case-insensitive globs
```

### Find Where a Function Is Defined

Enable extended debugging so `declare -F` includes the source file and line number, not just the function name:

```bash
shopt -s extdebug
declare -F <function>
```


## Useful Commands

### Functions

```bash
declare -F                      # List all function names
declare -F | grep function_name # Search for a specific function
declare -f quote_readline       # Display function definition
type quote_readline             # Alternative way to display definition
```

### Terminal and Keybindings

```bash
stty -a                         # Print terminal line settings
bind -p                         # List all keybindings
bind -p | grep '\\C'            # Bindings using Control key
bind -p | grep '\\C-a'          # Binding for Ctrl+A
```

### Bash Completion

```bash
yum install bash-completion
```

### Variables and Environment

```bash
set                             # List all variables (local + environment)
export                          # List only environment variables
echo $PS1                       # Display current prompt setting
```

### Running Commands Without Logging

```bash
uptime;history -d $(history 1)  # Run a command, then delete it from history
unset HISTFILE                  # Disable logging for entire session
```

### Running Remote Shell Scripts

```bash
# Process substitution: bash receives a file descriptor to read from
bash <(curl -s https://raw.githubusercontent.com/user/repo/master/install.sh)

# Command substitution: download script into a string, then execute it
bash -c "$(wget -qLO - https://example.com/script.sh)"

# Source into current shell (functions/variables persist in your session)
source <(curl -s https://example.com/functions.sh)

# Pipe directly to bash (most common on install guides)
curl -fsSL https://example.com/install.sh | bash

# Same with wget
wget -qO- https://example.com/install.sh | bash

# Pass arguments to the remote script
curl -fsSL https://example.com/install.sh | bash -s -- --arg1 --arg2

# Run as root
curl -fsSL https://example.com/install.sh | sudo bash

# Download first, inspect, then run (safer approach)
curl -fsSL https://example.com/install.sh -o install.sh && bash install.sh
```


## Prompt Customization

### Basic Custom Prompt

```bash
export PS1="[\[\e[1;32m\]\u@\[\e[1;33m\]\h\[\e[0m\] \W]\\$ "
```

### Git-Aware Prompt

Add to `~/.bash_profile`:

```bash
function parse_git_dirty {
  [[ $(git status 2> /dev/null | tail -n1) != "nothing to commit, working tree clean" ]] && echo "*"
}
function parse_git_branch {
  git branch --no-color 2> /dev/null | sed -e '/^[^*]/d' -e "s/* \(.*\)/(\1$(parse_git_dirty))/"
}

PS1='[\[\e[\033[1;32m\]\u@\[\e[1;33m\]\h \[\e[38;5;211m\]\W\[\e[\033[38;5;48m\] $(parse_git_branch)\[\e[\033[00m\]]\$ '
```

If you place the functions in `.bashrc`, you need to export them:

```bash
function parse_git_dirty {
  [[ $(git status 2> /dev/null | tail -n1) != "nothing to commit, working tree clean" ]] && echo "*"
}
export -f parse_git_dirty
```


## Bash History Configuration

### Per-Session or Per-Day History Files

```bash
mkdir /root/.history
```

Add to `~/.bash_profile`:

```bash
export HISTFILE=~/.history/histfile.$$              # per session
export HISTFILE=~/.history/$(date +%Y%m%d).hist     # per day
export HISTFILE=~/.history/$(date +%Y-%W).hist      # per week
export HISTTIMEFORMAT="%h/%d - %H:%M:%S "           # date + time
```

### Extended History Configuration

```bash
export HISTTIMEFORMAT='%Y-%m-%d %H:%M:%S - '
shopt -s histappend
HISTSIZE=100000
HISTFILESIZE=100000
HISTCONTROL=ignoreboth
HISTIGNORE='ls:bg:fg:history'
```

### Multi-User History with Timestamps

```bash
umask 027
HISTFILE="$HOME/.bash_history.$(logname)"
HISTTIMEFORMAT="%m/%d/%y %T "
PROMPT_COMMAND='history -a'
export LESS="-XF"
```

### Clear History

```bash
history -c > .bash_history
```


## Using sudo with Redirection

```bash
# WRONG: only echo runs as root, >> runs as your user
sudo echo "text" >> file

# CORRECT: tee runs as root and can write to the file
echo "text" | sudo tee -a file

# CORRECT: entire command including redirection runs as root
sudo bash -c 'echo "text" >> file'
```

### Appending Multi-Line Content

**Option 1: Use `sudo tee -a` (recommended)**

```bash
echo "
# Custom aliases
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
alias ..='cd ..'
" | sudo tee -a /root/.bashrc
```

**Option 2: Use `sudo sh -c`**

```bash
sudo sh -c 'cat >> /root/.bashrc << "EOF"

# Custom aliases
alias ll="ls -alF"
alias la="ls -A"
alias l="ls -CF"
alias ..="cd .."
EOF'
```

**Option 3: Edit directly**

```bash
sudo vim /root/.bashrc
```


## Modular .bashrc with .bashrc.d

Instead of a bloated `.bashrc`, split configuration into individual files:

```bash
mkdir ~/.bashrc.d
chmod 700 ~/.bashrc.d
```

Add to `~/.bashrc` or `~/.bash_profile`:

```bash
for file in ~/.bashrc.d/*.bashrc; do
  source "$file"
done
```

```bash
chmod +x ~/.bashrc.d/*.bashrc
```


## Script Collection Path

```bash
PATH=$PATH:$HOME/bin
export PATH
```


## Display Reminder Messages on Login

Add to `~/.bash_profile`:

```bash
if [ -r ./.to-do ]; then
  echo "++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++"
  echo "+REMINDER:"
  cat ./.to-do
  echo "++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++"
fi
```


## Useful One-Liners

```bash
# Check if SSH public key exists
[[ -f ~/.ssh/id_rsa.pub ]] && echo "SSH public key exists" || echo "SSH public key does not exist"

# Iterate over files and print their size
for file in *; do
  size=$(du -sh "$file" | cut -f1)
  echo "$file: $size"
done
```


## Keyboard Shortcuts

### CTRL Key Combinations

| Shortcut | Action |
|----------|--------|
| `CTRL+A` | Move to beginning of line |
| `CTRL+B` | Move backward one character |
| `CTRL+C` | Halt the current command |
| `CTRL+D` | Delete one character backward or log out |
| `CTRL+E` | Move to end of line |
| `CTRL+F` | Move forward one character |
| `CTRL+G` | Abort current editing command and ring bell |
| `CTRL+H` | Delete one character under cursor |
| `CTRL+J` | Same as RETURN |
| `CTRL+K` | Kill forward to end of line |
| `CTRL+L` | Clear screen and redisplay line |
| `CTRL+M` | Same as RETURN |
| `CTRL+N` | Next line in command history |
| `CTRL+O` | Same as RETURN, then display next history line |
| `CTRL+P` | Previous line in command history |
| `CTRL+Q` | Resume suspended shell output |
| `CTRL+R` | Search backward in history |
| `CTRL+S` | Search forward or suspend shell output |
| `CTRL+T` | Transpose two characters |
| `CTRL+U` | Kill backward to beginning of line |
| `CTRL+V` | Make next character typed verbatim |
| `CTRL+W` | Kill the word behind the cursor |
| `CTRL+X` | List possible filename completions |
| `CTRL+Y` | Yank (paste) last killed item |
| `CTRL+Z` | Stop current command (resume with `fg` or `bg`) |

### ALT Key Combinations

| Shortcut | Action |
|----------|--------|
| `ALT+B` | Move backward one word |
| `ALT+D` | Delete next word |
| `ALT+F` | Move forward one word |
| `ALT+H` | Delete one character backward |
| `ALT+T` | Transpose two words |
| `ALT+.` | Paste last word from last command (repeat to traverse history) |
| `ALT+U` | Uppercase from cursor to end of word |
| `ALT+L` | Lowercase from cursor to end of word |
| `ALT+C` | Capitalize letter under cursor, move to end of word |
| `ALT+R` | Revert changes to a command pulled from history |
| `ALT+?` | List possible completions |
| `ALT+^` | Expand line to most recent history match |

### Keyboard Macros and Editor

| Shortcut | Action |
|----------|--------|
| `CTRL+X` then `(` | Start recording a keyboard macro |
| `CTRL+X` then `)` | Finish recording keyboard macro |
| `CTRL+X` then `E` | Recall last recorded keyboard macro |
| `CTRL+X` then `CTRL+E` | Open current command in `$EDITOR`, execute on save |
| `CTRL+A` then `D` | Detach from screen session without killing it |


## References

- [GNU Bash Manual (PDF)](https://www.gnu.org/software/bash/manual/bash.pdf)
- [Bash Guide (wooledge.org)](http://mywiki.wooledge.org/BashGuide)
- [EzPrompt — PS1 Generator](https://ezprompt.net)
- [ShellCheck — Shell Script Linter](https://www.shellcheck.net)
- [Bash Hackers Wiki](https://bash-hackers.gabe565.com)
