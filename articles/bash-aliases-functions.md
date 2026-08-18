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
# Status and log
alias gs='git status -sb'
alias gl='git --no-pager log --oneline --graph --decorate -n 30'
alias gll='git --no-pager log --oneline --graph --decorate'
alias gd='git --no-pager diff'
alias gds='git diff --staged'

# Add/commit/push/pull
alias ga='git add'
alias gaa='git add -A'
alias gc='git commit'
alias gcm='git commit -m'
alias gca='git commit -a'
alias gcam='git commit -a -m'
alias gamend='git commit --amend'
alias gp='git push'
alias gpo='git push -u origin HEAD'
alias gpl='git pull --ff-only'
alias gf='git fetch'

# Branching
alias gb='git branch'
alias gba='git branch -a'
alias gbd='git branch -d'
alias gco='git checkout'
alias gcb='git checkout -b'
alias gsw='git switch'
alias gswn='git switch -c'
alias gcur='git rev-parse --abbrev-ref HEAD'

# Stash and clean
alias gst='git stash'
alias gstp='git stash pop'
alias gsl='git stash list'
alias gclean='git clean -xdf'

# Rebase helpers
alias grc='git rebase --continue'
alias gra='git rebase --abort'

# Show files ignored by git
alias gignored='git ls-files -i --exclude-standard'
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

## Kubernetes

```bash
if command -v kubectl >/dev/null 2>&1; then
  alias k='kubectl'
  alias kg='kubectl get'
  alias kd='kubectl describe'
  alias kdel='kubectl delete'
  alias kgp='kubectl get pods -A -o wide'
  alias kgs='kubectl get svc -A'
  alias kns='kubectl config set-context --current --namespace'
  alias kctx='kubectl config get-contexts'
  alias kcc='kubectl config current-context'
  alias kcu='kubectl config use-context'
fi
```

## Node.js and Package Managers

```bash
if command -v npm >/dev/null 2>&1; then
  alias ni='npm install'
  alias nid='npm install --save-dev'
  alias nig='npm install -g'
  alias nrd='npm run dev'
  alias nrb='npm run build'
  alias nrt='npm test'
  alias ns='npm start'
fi

if command -v yarn >/dev/null 2>&1; then
  alias y='yarn'
  alias yd='yarn dev'
  alias yb='yarn build'
  alias yt='yarn test'
fi

if command -v pnpm >/dev/null 2>&1; then
  alias p='pnpm'
  alias pd='pnpm dev'
  alias pb='pnpm build'
  alias pt='pnpm test'
fi
```

## Python

```bash
if command -v python3 >/dev/null 2>&1 || command -v python >/dev/null 2>&1; then
  alias py='python3 2>/dev/null || python'
  alias pipu='python3 -m pip install --upgrade pip'

  # Create virtualenv
  mkvenv() { python3 -m venv "${1:-.venv}"; }

  # Activate virtualenv (tries common paths and Windows)
  vact() {
    if [ -n "$1" ]; then
      source "$1/bin/activate" 2>/dev/null || source "$1/Scripts/activate" 2>/dev/null
    else
      source venv/bin/activate 2>/dev/null || \
      source .venv/bin/activate 2>/dev/null || \
      source venv/Scripts/activate 2>/dev/null || \
      source .venv/Scripts/activate 2>/dev/null
    fi
  }
fi
```

## Rust and Go

```bash
if command -v cargo >/dev/null 2>&1; then
  alias cb='cargo build'
  alias cr='cargo run'
  alias ct='cargo test'
  alias cc='cargo check'
fi

if command -v go >/dev/null 2>&1; then
  alias gbld='go build ./...'
  alias gtest='go test ./...'
  alias gtidy='go mod tidy'
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
