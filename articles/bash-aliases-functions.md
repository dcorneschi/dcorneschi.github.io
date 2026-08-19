# Bash Aliases and Functions

A curated set of practical aliases and helper functions for daily terminal work. Copy sections into `~/.bashrc` or `~/.bash_aliases` and reload with `source ~/.bashrc`.

## Portability Notes

- These are POSIX/Bash friendly and generally safe on macOS, Linux, WSL, and Git Bash
- Some commands (e.g., `ls` options) vary by platform — alternate definitions are included where applicable
- Aliases that depend on external tools are wrapped in `command -v` guards

## Setup

```bash
# Option 1: Add directly to ~/.bashrc
vi ~/.bashrc

# Option 2: Use a separate aliases file (recommended)
vi ~/.bash_aliases

# Make sure ~/.bashrc sources it:
[ -f ~/.bash_aliases ] && source ~/.bash_aliases
```

Reload after changes:

```bash
source ~/.bashrc
```

## Navigation and Quality of Life

```bash
# Safer rm/mv/cp (prompt before overwrite)
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# Quick directory jumps
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'
alias .....='cd ../../../..'

# List variants (GNU vs BSD/macOS)
if ls --color=auto >/dev/null 2>&1; then
  alias l='ls -lah --color=auto --group-directories-first'
  alias ll='ls -lAh --color=auto --group-directories-first'
  alias la='ls -A --color=auto --group-directories-first'
  alias lt='ls -ltr --color=auto'
else
  alias l='ls -lahG'
  alias ll='ls -lAhG'
  alias la='ls -AG'
  alias lt='ls -ltrG'
fi

# Colored grep by default
alias grep='grep --color=auto'
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'

# Human-friendly sizes
alias df='df -h'
alias du='du -h'

# Show PATH entries one per line
alias path='printf "%s\n" "$PATH" | tr : "\n"'

# Reload shell config (tries common locations)
alias reload='source ~/.bashrc 2>/dev/null || source ~/.bash_profile 2>/dev/null || source ~/.profile 2>/dev/null'

# Quick edit common files
alias ebash='${EDITOR:-nano} ~/.bashrc'
alias ealias='${EDITOR:-nano} ~/.bash_aliases'

# Misc
alias c='clear'
alias x='exit'
alias now='date +"%Y-%m-%d %H:%M:%S"'
alias today='date "+%Y-%m-%d"'
alias cal='cal -3'
alias count='wc -l'
```

## Shell Functions

### Create directory and cd into it

```bash
mkcd() { mkdir -p -- "$1" && cd -- "$1"; }
```

### Open current directory in file explorer (cross-platform)

```bash
open_here() {
  if command -v xdg-open >/dev/null 2>&1; then xdg-open . >/dev/null 2>&1 &
  elif command -v open >/dev/null 2>&1; then open .
  elif command -v explorer.exe >/dev/null 2>&1; then explorer.exe .
  else printf 'No known opener found.\n' >&2; fi
}
```

### Extract any archive format

```bash
extract() {
  if [ -f "$1" ]; then
    case "$1" in
      *.tar.bz2)  tar xjf "$1"   ;;
      *.tar.gz)   tar xzf "$1"   ;;
      *.tar.xz)   tar xJf "$1"   ;;
      *.tar.zst)  tar --use-compress-program=unzstd -xf "$1" ;;
      *.bz2)      bunzip2 "$1"   ;;
      *.rar)      unrar x "$1"   ;;
      *.gz)       gunzip "$1"    ;;
      *.tar)      tar xf "$1"    ;;
      *.tbz2)     tar xjf "$1"   ;;
      *.tgz)      tar xzf "$1"   ;;
      *.zip)      unzip "$1"     ;;
      *.7z)       7z x "$1"      ;;
      *.xz)       unxz "$1"      ;;
      *.Z)        uncompress "$1" ;;
      *)          echo "cannot extract '$1'" ;;
    esac
  else
    echo "'$1' is not a valid file"
  fi
}
```

### Find and kill process by name

```bash
killp() {
  ps aux | grep "$1" | grep -v grep | awk '{print $2}' | xargs kill -9
}
```

### Quick backup with timestamp

```bash
backup() {
  cp "$1"{,.backup-$(date +%Y%m%d_%H%M%S)}
}
```

### Find large files in current directory

```bash
bigfiles() {
  du -ah . 2>/dev/null | sort -h | tail -n 50
}
```

### Quick HTTP server with optional port

```bash
serve() {
  local port=${1:-8000}
  python3 -m http.server "$port" 2>/dev/null || python -m http.server "$port"
}
```

### Stopwatch

```bash
stopwatch() {
  date1=$(date +%s)
  read -rp "Press Enter to stop"
  date2=$(date +%s)
  echo $((date2 - date1)) 'seconds'
}
```

## History and Sudo Helpers

```bash
# Repeat last command with sudo
alias please='sudo $(fc -ln -1)'

# History search
alias h='history'
alias hg='history | grep'

# fzf-powered history search (if available)
if command -v fzf >/dev/null 2>&1; then
  hist() { fc -rl 1 | fzf --tac --no-sort --height 40% | sed 's/^ *[0-9]*\t//'; }
else
  hist() { history | less; }
fi
```

## Networking and Processes

```bash
alias myip='curl -s https://ifconfig.me || curl -s https://api.ipify.org'
alias localip='hostname -I'
alias ping='ping -c 5'

# Listening ports (ss preferred over netstat)
if command -v ss >/dev/null 2>&1; then
  alias ports='ss -tulpen'
else
  alias ports='netstat -tulpen'
fi

# Top alternative
if command -v htop >/dev/null 2>&1; then
  alias top='htop'
fi

# Process search
alias psg='ps aux | grep'

# Memory and CPU
alias free='free -h'
alias meminfo='cat /proc/meminfo'
alias cpuinfo='cat /proc/cpuinfo'
alias load='uptime'
```

## Search and Find

```bash
# Fast recursive search (ripgrep preferred)
if command -v rg >/dev/null 2>&1; then
  alias ag='rg -n --no-heading --color=always'
else
  alias ag='grep -Rin --color=always'
fi

# Find files by name
alias ff='find . -name'
alias ffi='find . -iname'
```

## Git

```bash
# Git
function gp() {
  git add -A && git commit -m "${1:-update}" && git push
}
```

```bash
# Simple function to update all Git repositories in current directory
update_all_repos() {
    echo "Updating all Git repositories in current directory..."
    for i in */.git; do
        if [[ -d "$i" ]]; then
            repo_name=$(basename "$(dirname "$i")")
            echo "Updating: $repo_name"
            (cd "$(dirname "$i")" && git pull)
        fi
    done
    echo "Done!"
}
```

## Docker

```bash
if command -v docker >/dev/null 2>&1; then
  alias d='docker'
  alias dc='docker compose'
  alias dps='docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'
  alias dpsa='docker ps -a'
  alias di='docker images'
  alias drm='docker rm'
  alias drmi='docker rmi'
  alias dstopall='docker stop $(docker ps -q) 2>/dev/null || true'
  alias drmfall='docker rm -f $(docker ps -aq) 2>/dev/null || true'
fi
```

```bash
# Function to enter container bash
dbash() {
    if [ $# -eq 1 ]; then
        if docker exec "$1" which bash >/dev/null 2>&1; then
            docker exec -it "$1" bash
        else
            docker exec -it "$1" sh
        fi
    else
        echo "Usage: dbash <container_name_or_id>"
    fi
}
```

## Kubernetes

```bash
if command -v kubectl >/dev/null 2>&1; then
  alias k='kubectl'
  alias kcontexts='kubectl config get-contexts'
  alias kctx='kubectl config use-context'
fi
```

## Service Management (systemd)

```bash
alias sctl='sudo systemctl'
alias sstart='sudo systemctl start'
alias sstop='sudo systemctl stop'
alias srestart='sudo systemctl restart'
alias sstatus='sudo systemctl status'
alias senable='sudo systemctl enable'
alias sdisable='sudo systemctl disable'
```

## Log Viewing

```bash
alias logs='sudo journalctl -f'
alias logauth='sudo tail -f /var/log/auth.log'
alias logapache='sudo tail -f /var/log/apache2/error.log'
alias lognginx='sudo tail -f /var/log/nginx/error.log'
```

## Package Management

```bash
# apt (Debian/Ubuntu)
alias aptup='sudo apt update && sudo apt upgrade'
alias aptin='sudo apt install'
alias aptrm='sudo apt remove'
alias aptsearch='apt search'

# yum/dnf (RHEL/CentOS)
alias yumup='sudo yum update'
alias yumin='sudo yum install'
```

## File Permissions and SSH

```bash
alias 644='chmod 644'
alias 755='chmod 755'
alias mx='chmod +x'
alias sshkey='cat ~/.ssh/id_rsa.pub'
alias sshgen='ssh-keygen -t ed25519'
```

## Web Servers

```bash
alias nginx-test='sudo nginx -t'
alias nginx-reload='sudo systemctl reload nginx'
alias apache-test='sudo apache2ctl configtest'
alias apache-reload='sudo systemctl reload apache2'
```

## Conditional Modern Replacements

Use better alternatives when available:

```bash
if command -v bat >/dev/null 2>&1; then
  alias cat='bat'
fi

if command -v exa >/dev/null 2>&1; then
  alias ls='exa'
  alias ll='exa -la'
  alias tree='exa --tree'
fi

if command -v fd >/dev/null 2>&1; then
  alias find='fd'
fi
```

## Tips

- Place aliases in `~/.bash_aliases` and source from `~/.bashrc` to keep things organized
- Keep aliases minimal and memorable — pick 5–10 you'll actually use daily
- Prefer functions when parameters are needed; use aliases for simple expansions
- Wrap tool-specific aliases in `command -v` guards for portability
- New shells auto-load if placed in the `~/.bashrc` or `~/.bash_profile` chain
