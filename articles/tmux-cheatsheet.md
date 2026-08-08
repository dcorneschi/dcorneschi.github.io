<p align="center">
  <img src="/articles/images/tmux-logo.svg" alt="tmux logo" width="300"/>
</p>

<h1 align="center">tmux Cheatsheet</h1>

Comprehensive tmux reference guide featuring installation instructions across multiple platforms, custom configuration examples, vim-style navigation, mouse support, and practical tips for productive terminal multiplexing.

## Installation

| Platform | Package Manager | Command |
|----------|----------------|---------|
| Ubuntu/Debian | apt | `sudo apt update && sudo apt install tmux` |
| Fedora/RHEL/CentOS | dnf | `sudo dnf install tmux` |
| Arch Linux | pacman | `sudo pacman -S tmux` |
| Alpine Linux | apk | `apk add tmux` |
| macOS | Homebrew | `brew install tmux` |
| macOS | MacPorts | `sudo port install tmux` |
| Windows (WSL) | apt/dnf/pacman | Use Linux commands above inside WSL |
| Windows (Git Bash/MSYS2) | pacman | `pacman -S tmux` |
| Windows | Chocolatey | `choco install tmux` |
| From Source | - | See instructions below |

### Building from Source

```bash
# Install dependencies (Ubuntu/Debian example)
sudo apt install libevent-dev ncurses-dev build-essential bison pkg-config

# Download and compile
wget https://github.com/tmux/tmux/releases/download/3.4/tmux-3.4.tar.gz
tar -xzf tmux-3.4.tar.gz
cd tmux-3.4
./configure
make
sudo make install
```

### Verify Installation

```bash
tmux -V
```

## Custom Configuration (Based on .tmux.conf)

### Prefix Key

| Command | Details |
|---------|---------|
| `Ctrl+a` | Prefix key (changed from default `Ctrl+b`) |

### Session Management

#### From Command Line

| Command | Details |
|---------|---------|
| `tmux` | Create new session with default session name |
| `tmux new` | Create new session with default session name |
| `tmux new-session` | Create new session with default session name |
| `tmux new -s session_name` | Create a new session called "session_name" |
| `tmux new -s session_name -n window` | Start session with named window |
| `tmux new-session -A -s session_name` | Attach to session or create if doesn't exist |
| `tmux ls` | List active tmux sessions |
| `tmux list-sessions` | List active tmux sessions |
| `tmux a` | Attach to last session |
| `tmux a -t session_name` | Attach to session by the name "session_name" |
| `tmux at -t session_name` | Attach to session by the name "session_name" |
| `tmux attach -t session_name` | Attach to session by the name "session_name" |
| `tmux attach-session -t session_name` | Attach to session by the name "session_name" |
| `tmux switch -t session_name` | Switch to session (when already inside tmux) |
| `tmux switch-client -t session_name` | Switch to session (when already inside tmux) |
| `tmux switch-client -n` | Switch to next session |
| `tmux switch-client -p` | Switch to previous session |
| `tmux kill-ses -t session_name` | Kill session by the name "session_name" |
| `tmux kill-session -t session_name` | Kill session by the name "session_name" |
| `tmux kill-session -a` | Kill all sessions except current |
| `tmux kill-session -a -t session_name` | Kill all sessions except "session_name" |
| `tmux kill-server` | Kill all sessions (terminate tmux server) |
| `tmux info` | Show session details (must be run inside tmux) |
| `tmux list-keys` | List all key bindings |
| `tmux list-commands` | List all tmux commands |
| `tmux list-windows -t session_name` | List all windows in session |
| `tmux list-panes -t session_name` | List all panes in session |

#### Inside Tmux

| Command | Details |
|---------|---------|
| `Ctrl+a d` | Detach from session, leaving it running in the background |
| `Ctrl+a D` | Choose which session to detach |
| `Ctrl+a s` | List and switch sessions (interactive) |
| `Ctrl+a w` | Session and window preview (interactive) |
| `Ctrl+a $` | Rename the session name |
| `Ctrl+a (` | Move to the previous session |
| `Ctrl+a )` | Move to the next session |
| `Ctrl+a :` | Enter command mode |
| `Ctrl+a :new` | Create new session from within tmux |
| `Ctrl+a :new -s session_name` | Create new named session from within tmux |
| `Ctrl+a :kill-session` | Kill current session |
| `Ctrl+a :attach -d` | Detach others on session (maximize window) |

### Session Sharing

#### From Command Line

| Command | Details |
|---------|---------|
| `tmux new -s shared` | Create a named session for sharing |
| `tmux attach -t shared` | Attach to shared session (multiple users can attach) |
| `tmux attach -t shared -r` | Attach in read-only mode |
| `tmux new -s shared -t existing` | Create grouped session (shares windows with existing) |
| `tmux list-clients` | List all clients attached to sessions |
| `tmux list-clients -t session_name` | List clients attached to specific session |
| `tmux detach -s session_name` | Detach all clients from session |
| `tmux detach-client -t client_id` | Detach specific client |

#### Inside Tmux

| Command | Details |
|---------|---------|
| `Ctrl+a :attach -d` | Detach other clients (exclusive access) |
| `Ctrl+a :list-clients` | Show all attached clients |
| `Ctrl+a :detach-client -t client_id` | Detach specific client |

#### Setting Up Session Sharing

For multiple users to share a session, they need access to the same tmux socket:

```bash
# User 1: Create shared session with group-writable socket
tmux -S /tmp/shared new -s shared
chmod 770 /tmp/shared

# User 2: Attach to shared session
tmux -S /tmp/shared attach -t shared

# Attach in read-only mode
tmux -S /tmp/shared attach -t shared -r
```

### Windows

#### From Command Line

| Command | Details |
|---------|---------|
| `tmux new-window` | Create new window in current session |
| `tmux new-window -n window_name` | Create new window with specific name |
| `tmux new-window -t session_name` | Create new window in specific session |
| `tmux new-window -t session_name:0` | Create new window at position 0 in session |
| `tmux list-windows` | List all windows in current session |
| `tmux list-windows -t session_name` | List all windows in specific session |
| `tmux select-window -t :0-9` | Select window by index |
| `tmux select-window -t window_name` | Select window by name |
| `tmux kill-window -t window_name` | Kill specific window |

#### Inside Tmux

| Command | Details |
|---------|---------|
| `Ctrl+a c` | Create new window |
| `Ctrl+a w` | List windows (interactive) |
| `Ctrl+a ,` | Rename current window |
| `Ctrl+a n` | Move to the next window |
| `Ctrl+a p` | Move to the previous window |
| `Ctrl+a f` | Find window by name |
| `Ctrl+a l` | Toggle last active window |
| `Ctrl+a 0-9` | Switch to window by number |
| `Ctrl+a &` | Kill/close current window |
| `Ctrl+a .` | Move window to another position |
| `Ctrl+a :swap-window -s 2 -t 1` | Swap window 2 with window 1 |
| `Ctrl+a :swap-window -t -1` | Move current window left by one position |
| `Ctrl+a :move-window -t number` | Move window to specific position |
| `Ctrl+a :move-window -r` | Renumber windows (remove gaps in sequence) |

### Panes

#### Splitting

| Command | Details |
|---------|---------|
| `Ctrl+a %` | Split pane vertically (left/right) |
| `Ctrl+a "` | Split pane horizontally (top/bottom) |
| `Ctrl+a :split-window -h` | Split horizontally (command) |
| `Ctrl+a :split-window -v` | Split vertically (command) |

#### Navigation (Vim-style)

| Command | Details |
|---------|---------|
| `Ctrl+a h` | Move to left pane |
| `Ctrl+a j` | Move to pane below |
| `Ctrl+a k` | Move to pane above |
| `Ctrl+a l` | Move to right pane |
| `Ctrl+a ;` | Toggle last active pane |
| `Ctrl+a Arrow keys` | Move to pane in direction |

#### Resizing

With prefix (repeatable):

| Command | Details |
|---------|---------|
| `Ctrl+a Left/Right/Up/Down` | Resize by 5 cells |
| `Ctrl+a :resize-pane -D/U/L/R cells` | Resize in direction by number of cells |

Without prefix:

| Command | Details |
|---------|---------|
| `Shift+Arrow` | Resize by 2 cells |
| `Ctrl+Arrow` | Resize by 2 cells |

#### Quick Layout Presets

| Command | Details |
|---------|---------|
| `Ctrl+a Alt+1` | Even horizontal splits |
| `Ctrl+a Alt+2` | Even vertical splits |
| `Ctrl+a Alt+3` | Horizontal span for main pane, vertical for others |
| `Ctrl+a Alt+4` | Vertical span for main pane, horizontal for others |
| `Ctrl+a Alt+5` | Tiled layout |

#### Other Pane Commands

| Command | Details |
|---------|---------|
| `Ctrl+a x` | Kill/close pane |
| `Ctrl+a z` | Toggle pane zoom (fullscreen) |
| `Ctrl+a o` | Cycle through panes |
| `Ctrl+a q` | Show pane numbers (type number to jump) |
| `Ctrl+a q 0-9` | Switch to pane by number |
| `Ctrl+a {` | Move pane left |
| `Ctrl+a }` | Move pane right |
| `Ctrl+a Space` | Toggle between pane layouts |
| `Ctrl+a !` | Break pane into new window |
| `Ctrl+a :join-pane -s 2 -t 1` | Join window 2 as pane in window 1 |
| `Ctrl+a :join-pane -s 2.1 -t 1.0` | Move pane 1 from window 2 to window 1 |
| `Ctrl+a :setw synchronize-panes` | Toggle sync input to all panes |

### Copy Mode (Vi-style)

#### Entering Copy Mode

| Command | Details |
|---------|---------|
| `Ctrl+a [` | Enter copy mode |
| `Ctrl+a PageUp` | Enter copy mode and scroll up one page |
| `PageUp` | Enter copy mode and scroll up |
| Mouse scroll up | Auto-enter copy mode |
| `q` | Quit copy mode |

#### Navigation in Copy Mode

| Command | Details |
|---------|---------|
| `h/j/k/l` | Move cursor left/down/up/right (Vim-style) |
| `w/b` | Jump word forward/backward |
| `0/$` | Start/end of line |
| `^` | Back to indentation |
| `g/G` | Go to top/bottom line |
| `H/M/L` | Move to top/middle/bottom of screen |
| `Ctrl+f/Ctrl+b` | Page down/up |
| `Ctrl+d/Ctrl+u` | Half page down/up |
| `f<char>` | Jump to character on line |
| `F<char>` | Jump backward to character on line |
| `/` | Search forward |
| `?` | Search backward |
| `n/N` | Next/previous search result |
| Arrow keys | Move up/down/left/right |

#### Selection and Copy

| Command | Details |
|---------|---------|
| `Space` | Begin selection (default) |
| `v` | Begin selection (custom vi binding) |
| `Enter` | Copy selection and exit copy mode |
| `y` | Copy selection and exit (custom vi binding) |
| `r` | Toggle rectangle selection (custom vi binding) |
| `Escape` | Clear selection |
| `p` | Paste buffer in copy mode |
| `Ctrl+a ]` | Paste buffer (outside copy mode) |

#### Paste from System Clipboard (without Shift)

When using tmux with mouse mode enabled, the terminal's normal paste is intercepted by tmux.
Here are ways to paste from your system clipboard depending on your setup:

**PuTTY:**

| Command | Details |
|---------|---------|
| `Shift + Right-click` | Bypasses tmux mouse capture, lets PuTTY paste directly |
| `Ctrl+a :set -g mouse off` | Temporarily disable mouse mode, right-click to paste, then re-enable |
| `Ctrl+a m` | Quick toggle mouse on/off (requires custom binding below) |

Add to `~/.tmux.conf`: `bind m set -g mouse \; display "Mouse: #{?mouse,ON,OFF}"`

**Linux (xclip/xsel):**

Add to `~/.tmux.conf`, then use `Ctrl+a v` to paste:

| Config | Details |
|--------|---------|
| `bind v run "xclip -selection clipboard -o \| tmux load-buffer - ; tmux paste-buffer"` | Using xclip |
| `bind v run "xsel --clipboard --output \| tmux load-buffer - ; tmux paste-buffer"` | Using xsel |

**macOS:**

| Config | Details |
|--------|---------|
| `bind v run "pbpaste \| tmux load-buffer - ; tmux paste-buffer"` | Paste from macOS clipboard |

**WSL/Windows Terminal:**

| Config | Details |
|--------|---------|
| `bind v run "powershell.exe -command 'Get-Clipboard' \| tmux load-buffer - ; tmux paste-buffer"` | Paste from Windows clipboard |

#### Buffer Management

| Command | Details |
|---------|---------|
| `Ctrl+a :show-buffer` | Display buffer_0 contents |
| `Ctrl+a :capture-pane` | Copy entire visible pane to buffer |
| `Ctrl+a :list-buffers` | Show all buffers |
| `Ctrl+a :choose-buffer` | Show all buffers and paste selected |
| `Ctrl+a :save-buffer file` | Save buffer contents to file |
| `Ctrl+a :delete-buffer -b 1` | Delete buffer_1 |

### Configuration

| Command | Details |
|---------|---------|
| `Ctrl+a r` | Reload tmux config |
| `tmux source-file ~/.tmux.conf` | Reload config from command line |
| `tmux show-options -g` | Show all global options |
| `tmux show-options -gw` | Show all global window options |
| `tmux show-options -s` | Show all server options |
| `tmux show-options -g option_name` | Show specific global option value |
| `tmux show-environment` | Show environment variables |
| `tmux show-environment -g` | Show global environment variables |

#### Creating Custom Configuration

Create or edit `~/.tmux.conf` to customize tmux settings:

```bash
# Example: Change prefix key
unbind C-b
set -g prefix C-a

# Enable mouse support
set -g mouse on

# Set vi mode
setw -g mode-keys vi
```

### Mouse Support

| Feature | Details |
|---------|---------|
| Click | Select pane |
| Drag pane borders | Resize panes |
| Scroll | Navigate history (auto-enters copy mode) |

#### Copying Text with Mouse

| Command | Details |
|---------|---------|
| Hold `Shift` + Double-click | Select word |
| Hold `Shift` + Click and drag | Select text |
| `Ctrl+Shift+C` | Copy (or `Shift + Right-click → Copy`) |
| `Ctrl+Shift+V` | Paste (or `Shift + Right-click → Paste`) |

Holding `Shift` bypasses tmux mouse mode and brings back OS default behavior (useful when pasting from system clipboard).

#### Toggle Mouse Mode

| Command | Details |
|---------|---------|
| `Ctrl+a :set -g mouse off` | Disable mouse mode (for easier text selection) |
| `Ctrl+a :set -g mouse on` | Re-enable mouse mode |

### Status Bar

| Setting | Details |
|---------|---------|
| Position | Top of screen |
| Content | Session name, window list, hostname, time |
| Active window | Red background |
| Inactive windows | Blue background |

### Scrollback

| Setting | Details |
|---------|---------|
| History limit | 10,000 lines |
| `PageUp/PageDown` | Navigate history |

## Common Tmux Commands (General)

### Useful Commands

| Command | Details |
|---------|---------|
| `:set -g option` | Set option for all sessions |
| `:setw -g option` | Set option for all windows |
| `:set mouse on` | Enable mouse mode |
| `:setw -g mode-keys vi` | Use vi keys in copy mode |
| `:setw synchronize-panes on` | Sync input to all panes |
| `:setw synchronize-panes off` | Disable sync |
| `:swap-window -s src -t dst` | Swap windows |
| `:move-window -t number` | Move window to position |
| `:resize-pane -D` | Resize pane down |
| `:resize-pane -U` | Resize pane up |
| `:resize-pane -L` | Resize pane left |
| `:resize-pane -R` | Resize pane right |
| `:resize-pane -D 20` | Resize pane down by 20 cells |
| `:resize-pane -t 2 -L 20` | Resize specific pane by ID |
| `:list-keys` | List all key bindings |

### Help

| Command | Details |
|---------|---------|
| `Ctrl+a ?` | List all keybindings (interactive help) |
| `q` | Exit help screen |
| `Ctrl+s` | Search forward in help screen |
| `n` | Next search result in help |
| `N` | Previous search result in help |
| `/` | Search forward (vi-style) |
| `?` | Search backward (vi-style) |

### Tips

| Command / Tip | Details |
|---------------|---------|
| `Ctrl+a ?` | List all keybindings (press `q` to exit help) |
| `Ctrl+a t` | Show big clock |
| `Ctrl+a z` | Zoom pane (useful for temporarily focusing on one pane) |
| `Ctrl+a Space` | Cycle through different pane layouts |
| `:setw synchronize-panes` | Type commands in all panes simultaneously (great for multiple servers) |
| `:setw synchronize-panes off` | Toggle synchronize off |
| All resize bindings | Repeatable (hold prefix and repeat arrow keys) |
| Copy mode | Uses vi keybindings for navigation |
| Mouse support | Works in modern terminals |
| Window auto-rename | Tmux renames windows based on running programs (e.g., `top`, `vim`) |
| Sessions persist | Even after detaching — perfect for long-running tasks |
| Mental model | Sessions = Notebooks, Windows = Chapters, Panes = Pages |
| `Alt+1` through `Alt+5` | Quick layout presets to organize messy pane arrangements |

### Resources

* [https://github.com/tmux/tmux](https://github.com/tmux/tmux)
* [https://github.com/rothgar/awesome-tmux](https://github.com/rothgar/awesome-tmux)
* [https://pragprog.com/titles/bhtmux2/tmux-2](https://pragprog.com/titles/bhtmux2/tmux-2)
