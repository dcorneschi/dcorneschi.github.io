# Bash History Guide

Complete reference for working with command history in Bash. Covers viewing, searching, recalling, modifying, and configuring history — from basic recall to advanced event designators and word designators.

## Viewing and Managing History

### Basic Commands

```bash
# Display full history
history

# Show only the last 20 commands
history 20

# Clear the current session history
history -c

# Delete a specific entry by line number
history -d 42

# Delete a range of entries (bash 5.0+)
history -d 10-20

# Write current session history to the history file (overwrites)
history -w

# Append current session history to the history file (without overwriting)
history -a

# Read the history file and append its contents to the current session
history -r

# Read new entries from the history file (added by other sessions)
history -n

# Perform history substitution on args and print (without storing or executing)
history -p '!!'

# Add a command to the history list without executing it
history -s "long-command --with-flags that-I-want-to-recall-later"
```

### History File

By default, Bash stores history in `~/.bash_history`. The file is written when a session exits (or when you run `history -w` / `history -a`).

```bash
# Show the history file location
echo $HISTFILE

# Show how many lines the history file retains
echo $HISTFILESIZE

# Show how many commands Bash keeps in memory for the session
echo $HISTSIZE
```

## Recalling Commands

### Event Designators

Event designators let you reference commands from the history list:

| Designator | Description |
|-----------|-------------|
| `!!` | The last command (the entire line) |
| `!n` | Command number `n` from history |
| `!-n` | The command `n` lines back (e.g., `!-2` = two commands ago) |
| `!string` | Most recent command that starts with `string` |
| `!?string?` | Most recent command that contains `string` |
| `!#` | The current command line typed so far |

### Common Patterns

```bash
# Re-run the last command with sudo
sudo !!

# Re-run the last command that started with "git"
!git

# Re-run the last command that contained "deploy"
!?deploy?

# Run command number 154 from history
!154

# Run the command from 3 entries ago
!-3
```

## Word Designators

Word designators select specific words (arguments) from a history entry. They follow the event designator, separated by a colon:

```
!event:word_designator
```

| Designator | Description |
|-----------|-------------|
| `0` | The command name (zeroth word) |
| `n` | The nth argument (1-indexed) |
| `^` | The first argument (same as `:1`) |
| `$` | The last argument |
| `%` | The first word matched by the most recent `!?string?` search |
| `*` | All arguments (everything except word 0) |
| `x-y` | Range of words from `x` to `y` (`-y` abbreviates `0-y`) |
| `x*` | Words from `x` to the last (abbreviates `x-$`) |
| `x-` | Words from `x` to the second-to-last (drops the last word) |

### Examples

```bash
# Given previous command: cp /var/log/syslog /tmp/syslog.bak

# Get the last argument of the previous command
echo !!:$
# → /tmp/syslog.bak

# Get the first argument of the previous command
echo !!:^
# → /var/log/syslog

# Get all arguments of the previous command
echo !!:*
# → /var/log/syslog /tmp/syslog.bak

# Get the second argument of the previous command
echo !!:2
# → /tmp/syslog.bak
```

### Shortcuts for the Last Command

These shortcuts don't require the `!!:` prefix:

| Shortcut | Equivalent | Description |
|----------|-----------|-------------|
| `!$` | `!!:$` | Last argument of the previous command |
| `!^` | `!!:^` | First argument of the previous command |
| `!*` | `!!:*` | All arguments of the previous command |

```bash
# Create a directory, then cd into it
mkdir /opt/myapp
cd !$
# → cd /opt/myapp

# View a file, then edit it
cat /etc/nginx/nginx.conf
vim !$
# → vim /etc/nginx/nginx.conf

# List files, then use the same path
ls /var/log/nginx/
cd !$
# → cd /var/log/nginx/
```

### Alt+. (Meta-dot) Shortcut

Press `Alt+.` (or `Esc` then `.`) to insert the last argument of the previous command at the cursor. Press it repeatedly to cycle through last arguments of earlier commands.

## Modifiers

Modifiers alter the result of a history expansion. They follow word designators, separated by a colon:

```
!event:word:modifier
```

| Modifier | Description |
|----------|-------------|
| `h` | Remove the trailing filename, keep the head (dirname) |
| `t` | Remove the leading path, keep the tail (basename) |
| `r` | Remove the trailing suffix (`.ext`) |
| `e` | Remove everything except the trailing suffix |
| `p` | Print the result but don't execute it |
| `q` | Quote the substituted words, escaping further substitutions |
| `x` | Quote and break into words at spaces, tabs, and newlines |
| `s/old/new/` | Substitute `old` with `new` (first occurrence) |
| `gs/old/new/` | Substitute `old` with `new` (all occurrences globally) |
| `G` | Apply the following `s` or `&` modifier once to each word |
| `&` | Repeat the previous substitution |

### Examples

```bash
# Given previous command: tar -czf /backups/app-2024.tar.gz /opt/app/

# Get the directory part of the last argument
echo !!:$:h
# → /opt/app

# Get the filename part of argument 2
echo !!:2:t
# → app-2024.tar.gz

# Remove file extension from the last argument
echo !!:2:r
# → /backups/app-2024.tar

# Print expansion without executing (useful for verification)
!git:p
# Prints the command but doesn't run it
```

### Quick Substitution

The `^old^new^` syntax is a shortcut for `!!:s/old/new/`:

```bash
# Ran with a typo
cat /etc/ngingx/nginx.conf
# bash: /etc/ngingx/nginx.conf: No such file or directory

# Fix it quickly
^ngingx^nginx^
# → cat /etc/nginx/nginx.conf

# Change a flag
systemctl start nginx
^start^restart^
# → systemctl restart nginx

# Change a filename
vim config.yml
^yml^yaml^
# → vim config.yaml
```

## Searching History

### Interactive Search (Ctrl+R)

Press `Ctrl+R` to start a reverse incremental search. Type part of a command and Bash finds the most recent match:

```
(reverse-i-search)`deploy': ansible-playbook deploy.yml -i production
```

| Key | Action |
|-----|--------|
| `Ctrl+R` | Start reverse search / find next match going backward |
| `Ctrl+S` | Forward search (find next match going forward) |
| `Enter` | Execute the found command |
| `Ctrl+G` | Cancel search, return to prompt |
| `Ctrl+J` | Accept the found command onto the command line (without executing) |
| `Tab` | Accept and allow editing before execution |
| `Esc` | Accept the found command for editing (same as `Ctrl+J`) |

### grep Through History

```bash
# Search for all commands containing "docker"
history | grep docker

# Search with line numbers
history | grep -n "git push"

# Last 5 commands matching a pattern
history | grep ssh | tail -5

# Regex search
history | grep -E "git (commit|push|pull)"
```

### fc Command (Fix Command)

The `fc` built-in opens commands in your editor for modification before re-execution:

```bash
# Open the last command in $FCEDIT (or $EDITOR, or vi)
fc

# Open command number 100 in editor
fc 100

# Open commands 100 through 110 in editor
fc 100 110

# Open the last command starting with "git" in editor
fc git

# Use a specific editor
fc -e nano

# List the last 16 commands (default range)
fc -l

# List the last 20 commands
fc -l -20

# List commands with no line numbers
fc -ln -20

# List in reverse order
fc -lr -20

# Re-execute the last command (equivalent to `r` alias)
fc -s

# Re-execute the last command starting with "git"
fc -s git

# Replace "test" with "prod" and re-execute the last matching command
fc -s test=prod

# Replace in the last command starting with "ansible"
fc -s staging=production ansible
```

**Useful alias:**

```bash
# r='fc -s' — so 'r' re-runs last command, 'r git' re-runs last git command
alias r='fc -s'
```

## Configuration

### Key Variables

Set these in `~/.bashrc` or `~/.bash_profile`:

```bash
# Number of commands stored in memory during a session
HISTSIZE=10000

# Number of lines stored in the history file
HISTFILESIZE=20000

# Location of the history file
HISTFILE=~/.bash_history

# Commands to exclude from history
# ignorespace  — commands starting with a space are not saved
# ignoredups   — consecutive duplicate commands are not saved
# ignoreboth   — combines ignorespace and ignoredups
# erasedups    — remove all previous occurrences of a command before saving
HISTCONTROL=ignoreboth:erasedups

# Patterns to exclude from history (colon-separated)
HISTIGNORE="ls:cd:pwd:exit:clear:history"

# Timestamp format for history entries
HISTTIMEFORMAT="%F %T  "
# Output: 1042  2024-03-15 14:23:07  git push origin main
```

### Recommended Configuration

```bash
# ~/.bashrc — history settings

# Large history
HISTSIZE=50000
HISTFILESIZE=100000

# Avoid duplicates and commands starting with spaces
HISTCONTROL=ignoreboth:erasedups

# Ignore trivial commands
HISTIGNORE="ls:ll:cd:pwd:exit:clear:history:bg:fg"

# Add timestamps
HISTTIMEFORMAT="%F %T  "

# Append to history file instead of overwriting
shopt -s histappend

# Save multi-line commands as a single entry
shopt -s cmdhist

# Immediately append each command to history file (share across sessions)
PROMPT_COMMAND="history -a; history -n; $PROMPT_COMMAND"

# Re-edit a failed history substitution rather than executing it
shopt -s histreedit

# Verify history expansion before executing (show it first)
shopt -s histverify
```

### Shell Options (shopt)

| Option | Description |
|--------|-------------|
| `histappend` | Append to `~/.bash_history` on exit instead of overwriting |
| `histverify` | Show expanded history command for review before executing |
| `histreedit` | Allow re-editing if a history substitution fails |
| `cmdhist` | Save multi-line commands as a single history entry |
| `lithist` | Save multi-line commands with embedded newlines (requires `cmdhist`) |

```bash
# Enable an option
shopt -s histappend

# Disable an option
shopt -u histverify

# Check if an option is set
shopt histappend
```

## Sharing History Across Sessions

By default, each terminal session has its own history in memory and only writes to `~/.bash_history` on exit. To share history in real time:

```bash
# In ~/.bashrc
shopt -s histappend
PROMPT_COMMAND="history -a; history -n; $PROMPT_COMMAND"
```

How it works:
- `history -a` — appends the last command to the history file after each prompt
- `history -n` — reads new entries from the file that were written by other sessions

For a more aggressive approach (full sync on every command):

```bash
PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"
```

This writes, clears memory, and re-reads the full file — giving you a unified history view across all terminals at the cost of losing per-session ordering.

## Preventing Commands from Being Saved

```bash
# Method 1: Start the command with a space (requires HISTCONTROL=ignorespace or ignoreboth)
 export SECRET_KEY="abc123"

# Method 2: Disable history temporarily
set +o history
sensitive-command --password=secret
set -o history

# Method 3: Use HISTIGNORE for patterns
HISTIGNORE="*password*:*secret*:*token*"
```

## Useful Patterns

### Run the Last Command as Root

```bash
sudo !!
```

### Reuse Arguments in New Commands

```bash
# Download a file, then extract it
wget https://example.com/archive.tar.gz
tar xzf !$

# Create a directory and cd into it (function)
mkcd() { mkdir -p "$1" && cd "$1"; }

# Alternative using history
mkdir -p /opt/project/src
cd !$
```

### Fix Typos Quickly

```bash
# Typo in the command
git psuh origin main
^psuh^push^
# → git push origin main
```

### Repeat Commands with Modifications

```bash
# Deploy to staging
ansible-playbook deploy.yml -i staging

# Now deploy to production (substitute staging → production)
!!:gs/staging/production/
# → ansible-playbook deploy.yml -i production
```

### Print Before Executing

```bash
# See what the expansion would produce (without running it)
!docker:p
# Prints: docker compose up -d (or whatever the last docker command was)

# If it looks right, run it
!docker
```

### Delete Sensitive Commands from History

```bash
# Find the command number
history | grep password
#  1042  mysql -u root -pMySecret123 mydb

# Delete that specific entry
history -d 1042

# Write the updated history to file
history -w
```

## Quick Reference

| Want to... | Use |
|-----------|-----|
| Re-run last command | `!!` |
| Re-run last command as root | `sudo !!` |
| Re-run last command starting with X | `!X` |
| Get last argument of previous command | `!$` or `Alt+.` |
| Get first argument of previous command | `!^` |
| Get all arguments of previous command | `!*` |
| Get Nth argument of previous command | `!!:N` |
| Fix a typo in the last command | `^old^new^` |
| Global substitution on last command | `!!:gs/old/new/` |
| Search history interactively | `Ctrl+R` |
| Search forward in history | `Ctrl+S` |
| Print expansion without running | `!command:p` |
| Clear session history | `history -c` |
| Delete entry N | `history -d N` |
| Delete range | `history -d 10-20` |
| Save history now (overwrite) | `history -w` |
| Append to history now | `history -a` |
| Reload history from file | `history -r` |
| Add command to history without executing | `history -s "command"` |
| Test expansion without executing | `history -p '!!'` |
| Avoid saving a command | Prefix with a space |
| Share history across terminals | `PROMPT_COMMAND="history -a; history -n"` |
| Add timestamps to history | `HISTTIMEFORMAT="%F %T  "` |
| Open last command in editor | `fc` |
| Edit and re-run command N | `fc N` |
