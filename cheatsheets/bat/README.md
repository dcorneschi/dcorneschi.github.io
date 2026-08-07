# bat Cheatsheet

`bat` is a `cat` clone with syntax highlighting, git integration, and automatic paging. It supports a wide range of languages and integrates with other tools via piping.

---

## Installation

```bash
# Debian / Ubuntu
sudo apt install bat
# Binary is installed as 'batcat' — create an alias:
alias bat='batcat'

# RHEL / Fedora
sudo dnf install bat

# macOS
brew install bat

# Cargo (Rust)
cargo install bat
```

> **Note:** On Debian/Ubuntu the binary is called `batcat` due to a naming conflict. Use `alias bat='batcat'` in your shell profile or create a symlink: `ln -s /usr/bin/batcat ~/.local/bin/bat`

---

## Basic Usage

```bash
# Display a file with syntax highlighting
bat file.py

# Display multiple files
bat file1.py file2.py

# Concatenate files (like cat)
bat file1.txt file2.txt > combined.txt

# Read from stdin
echo '{"key": "value"}' | bat -l json
curl -s https://example.com/api | bat -l json

# Display with line numbers only (no other decorations)
bat -n file.py

# Display without any decorations (plain output, like cat)
bat -p file.py
bat --plain file.py
```

---

## Syntax Highlighting

```bash
# Explicit language (override auto-detection)
bat -l python script.sh
bat --language yaml config.txt

# List all supported languages
bat --list-languages

# Force a file extension mapping
bat -l Dockerfile myfile
```

---

## Themes

```bash
# List all available themes
bat --list-themes

# Use a specific theme
bat --theme gruvbox-dark file.py
bat --theme "Monokai Extended" file.py

# Preview all themes on a file
bat --list-themes | fzf --preview="bat --theme={} --color=always file.py"

# Set default theme via environment variable
export BAT_THEME="gruvbox-dark"
```

---

## Line Ranges

```bash
# Show only lines 10 to 20
bat --line-range 10:20 file.py
bat -r 10:20 file.py

# Show from line 30 to end
bat -r 30: file.py

# Show first 10 lines
bat -r :10 file.py

# Multiple ranges
bat -r 1:5 -r 20:25 file.py
```

---

## Line Highlighting

```bash
# Highlight specific lines
bat --highlight-line 5 file.py
bat -H 5 file.py

# Highlight a range
bat -H 10:15 file.py

# Multiple highlights
bat -H 3 -H 10:12 -H 20 file.py
```

---

## Decorations and Style

```bash
# Show only line numbers (no grid, no header)
bat -n file.py
bat --number file.py

# Plain mode — no decorations at all
bat -p file.py
bat --plain file.py

# Choose specific components
bat --style=numbers,changes file.py
bat --style=header,grid file.py
bat --style=full file.py

# Available style components:
#   auto, full, plain, header, header-filename, header-filesize,
#   grid, rule, numbers, snip, changes
```

---

## Paging

```bash
# Force paging (even for short files)
bat --paging=always file.py

# Disable paging (print directly)
bat --paging=never file.py
bat -P file.py

# Set pager (default is less)
bat --pager="less -RF" file.py

# Environment variable
export BAT_PAGER="less -RF"
```

---

## Git Integration

`bat` shows git modifications in the left margin by default:

| Marker | Meaning |
|--------|---------|
| `~` | Modified line |
| `+` | Added line |
| `-` | Deleted line |

```bash
# Show git diff context
bat --diff file.py

# Disable git decorations
bat --style=plain file.py
```

---

## Piping and Integration

```bash
# Use as a pager for git
git diff | bat -l diff
git show HEAD | bat -l diff
git log -p | bat -l diff

# Use as a man pager
export MANPAGER="sh -c 'col -bx | bat -l man -p'"
man ls

# Use with find
find . -name "*.py" -exec bat {} +

# Use with tail
tail -f /var/log/syslog | bat --paging=never -l syslog

# Use with fzf preview
fzf --preview="bat --color=always --style=numbers {}"

# Use with ripgrep
rg --files | fzf --preview="bat --color=always {}"
```

---

## Configuration File

Create `~/.config/bat/config` (or `$BAT_CONFIG_PATH`) for persistent settings:

```bash
# Generate default config
bat --generate-config-file

# Edit config
vim ~/.config/bat/config
```

Example `~/.config/bat/config`:

```
# Set the theme
--theme="gruvbox-dark"

# Show line numbers
--style="numbers,changes,header"

# Use italic text
--italic-text=always

# Set pager
--pager="less -RF"

# Map file types
--map-syntax "*.conf:INI"
--map-syntax ".ignore:Git Ignore"
```

---

## Custom File Type Mappings

```bash
# Map an extension to a language
bat --map-syntax "*.ino:C++" file.ino
bat --map-syntax "*.conf:INI" myapp.conf
bat --map-syntax ".env:Bourne Again Shell (bash)" .env

# Map a filename glob
bat --map-syntax "Jenkinsfile:Groovy" Jenkinsfile
```

---

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `BAT_THEME` | Default theme |
| `BAT_STYLE` | Default style components |
| `BAT_PAGER` | Pager command |
| `BAT_CONFIG_PATH` | Path to config file |
| `BAT_TABS` | Tab width (default: 4) |

```bash
# Add to ~/.bashrc or ~/.zshrc
export BAT_THEME="gruvbox-dark"
export BAT_STYLE="numbers,changes"
export BAT_PAGER="less -RF"
export BAT_TABS=4
```

---

## Useful Aliases

```bash
# Replace cat
alias cat='bat --paging=never'

# Quick preview (no paging, no decorations)
alias bp='bat -pp'

# Bat with line numbers only
alias bn='bat -n'

# Preview YAML/JSON
alias byaml='bat -l yaml'
alias bjson='bat -l json'
```

---

## Quick Reference

| Command | Description |
|---------|-------------|
| `bat file` | Display file with syntax highlighting |
| `bat -n file` | Show with line numbers only |
| `bat -p file` | Plain output (like cat, with highlighting) |
| `bat -pp file` | Plain output, no paging |
| `bat -l json file` | Force language |
| `bat -r 10:20 file` | Show line range |
| `bat -H 5 file` | Highlight a line |
| `bat --diff file` | Show git changes |
| `bat --list-languages` | List supported languages |
| `bat --list-themes` | List available themes |
| `bat --theme name file` | Use a specific theme |
| `bat --generate-config-file` | Create config file |
