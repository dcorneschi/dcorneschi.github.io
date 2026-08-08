<p align="center">
  <img src="/articles/images/kitty-logo.svg" alt="Kitty Logo" width="200"/>
</p>

<h1 align="center">Kitty Cheatsheet</h1>

Comprehensive kitty reference guide featuring installation instructions, keyboard shortcuts, layout configurations, and practical tips for productive terminal usage.

## Installation

| Platform | Package Manager | Command |
|----------|----------------|---------|
| macOS | Homebrew | `brew install kitty` |
| Ubuntu/Debian | apt | `sudo apt install kitty` |
| Fedora/RHEL | dnf | `sudo dnf install kitty` |
| Arch Linux | pacman | `sudo pacman -S kitty` |
| Alpine Linux | apk | `apk add kitty` |
| Linux (binary) | - | `curl -L https://sw.kovidgoyal.net/kitty/installer.sh \| sh /dev/stdin` |

### Verify Installation
```bash
kitty --version
```

## Quick Access Terminal (macOS)

Create a Quake-style quick access terminal that appears/disappears with <code>Ctrl+`</code>:

1. Run this command in kitty to create the quick access window:
```bash
kitten quick-access-terminal
```

2. The window will appear at the top of your screen. Close it by running the command again or pressing `Ctrl+D`

3. Set up the keyboard shortcut:
   - Go to System Settings → Keyboard → Keyboard Shortcuts → Services → General
   - Find "Quick access to kitty" entry
   - Click and set your shortcut to <code>Ctrl+`</code>

4. Now press <code>Ctrl+`</code> anytime to show/hide your quick access terminal

### Configuration

Customize the quick access terminal by creating `~/.config/kitty/quick-access-terminal.conf`:

```
# Window size
lines 20
columns 120

# Position (top, bottom, left, right, center, background)
edge top

# Appearance
background_opacity 0.95
hide_on_focus_loss yes

# Override kitty settings for this window
kitty_override font_size=14
kitty_override background=#1e1e1e
```

See the [official documentation](https://sw.kovidgoyal.net/kitty/kittens/quick-access-terminal/) for more options.

## Core Concepts

| Concept | Description |
|---------|-------------|
| **OS Window** | Top-level window containing tabs |
| **Tab** | Container for multiple windows |
| **Window** | Individual terminal pane within a tab |
| **Layout** | Arrangement pattern for windows |

## Tabs

| Action | Shortcut | macOS |
|--------|----------|-------|
| New tab | `Ctrl+Shift+T` | `Cmd+T` |
| Close tab | `Ctrl+Shift+Q` | `Cmd+W` |
| Next tab | `Ctrl+Shift+Right` | `Shift+Cmd+]` |
| Previous tab | `Ctrl+Shift+Left` | `Shift+Cmd+[` |
| Move tab forward | `Ctrl+Shift+.` | `Ctrl+Shift+.` |
| Move tab backward | `Ctrl+Shift+,` | `Ctrl+Shift+,` |
| Set tab title | `Ctrl+Shift+Alt+T` | `Shift+Cmd+I` |

To add shortcuts for jumping to specific tabs (no default), add to your config:
```
# Map cmd + <num> to corresponding tabs
map cmd+1 goto_tab 1
map cmd+2 goto_tab 2
map cmd+3 goto_tab 3
map cmd+4 goto_tab 4
map cmd+5 goto_tab 5
map cmd+6 goto_tab 6
map cmd+7 goto_tab 7
map cmd+8 goto_tab 8
map cmd+9 goto_tab 9
```

To display tab numbers in the tab bar:
```
# Show tab number before title (e.g., "1: zsh")
tab_title_template "{index}: {title}"
```

### Tab Bar Styles

| Style | Preview | Configuration |
|-------|---------|---------------|
| **fade** | <pre>│ zsh ░░ vim ░░ htop │</pre> | `tab_bar_style fade` |
| **separator** | <pre>│ zsh │ vim │ htop │</pre> | `tab_bar_style separator` |
| **powerline** | <pre>▌zsh▶▌vim▶▌htop▶</pre> | `tab_bar_style powerline` |
| **slant** | <pre>╱ zsh ╱ vim ╱ htop ╱</pre> | `tab_bar_style slant` |
| **custom** | User-defined template | `tab_bar_style custom` |
| **hidden** | (no tab bar) | `tab_bar_style hidden` |

```
# Slanted tab separators
tab_bar_style slant
```

## Windows

### Basic Operations

| Action | Shortcut | macOS Default |
|--------|----------|---------------|
| New window | `Ctrl+Shift+Enter` | `Cmd+Enter` |
| Close window | `Ctrl+Shift+W` | `Shift+Cmd+D` |
| Next window | `Ctrl+Shift+]` | `Ctrl+Shift+]` |
| Previous window | `Ctrl+Shift+[` | `Ctrl+Shift+[` |
| Move window forward | `Ctrl+Shift+F` | `Ctrl+Shift+F` |
| Move window backward | `Ctrl+Shift+B` | - |
| Show window size | - | `Ctrl+Shift+B` |
| Move window to top | <code>Ctrl+Shift+`</code> | <code>Ctrl+Shift+`</code> |

### Window Navigation (Custom Mappings)

```
# Navigate between windows directionally (no default)
map ctrl+shift+left neighboring_window left
map ctrl+shift+right neighboring_window right
map ctrl+shift+up neighboring_window top
map ctrl+shift+down neighboring_window bottom

# Move windows directionally (no default)
map shift+left move_window left
map shift+right move_window right
map shift+up move_window top
map shift+down move_window bottom
```

### Window Resizing

| Action | Shortcut | macOS |
|--------|----------|-------|
| Enter resize mode | `Ctrl+Shift+R` | `Cmd+R` |

In resize mode, press `w` (wider), `n` (narrower), `t` (taller), `s` (shorter), or `Esc` to exit.

To resize with arrow keys, add these mappings to your `~/.config/kitty/kitty.conf`:
```
# Resize windows with arrow keys
map ctrl+shift+left resize_window narrower
map ctrl+shift+right resize_window wider
map ctrl+shift+up resize_window taller
map ctrl+shift+down resize_window shorter
```

These shortcuts work both in and out of resize mode.

## Layouts

| Layout | Diagram | Configuration |
|--------|---------|---------------|
| **Stack** | <pre>┌──────────────────┐<br>│                  │<br>│                  │<br>│                  │<br>└──────────────────┘</pre> | `enabled_layouts stack` |
| **Tall** | <pre>┌────────┬─────────┐<br>│        │         │<br>│        ├─────────┤<br>│        │         │<br>└────────┴─────────┘</pre> | `enabled_layouts tall:bias=50;full_size=1;mirrored=false` |
| **Fat** | <pre>┌──────────────────┐<br>│                  │<br>├────────┬─────────┤<br>│        │         │<br>└────────┴─────────┘</pre> | `enabled_layouts fat:bias=50;full_size=1;mirrored=false` |
| **Grid** | <pre>┌────────┬─────────┐<br>│        │         │<br>├────────┼─────────┤<br>│        │         │<br>└────────┴─────────┘</pre> | `enabled_layouts grid` |
| **Splits** | <pre>┌────────┬─────────┐<br>│        │         │<br>│        ├────┬────┤<br>│        │    │    │<br>└────────┴────┴────┘</pre> | `enabled_layouts splits:split_axis=horizontal` |
| **Horizontal** | <pre>┌─────┬──────┬─────┐<br>│     │      │     │<br>│     │      │     │<br>│     │      │     │<br>└─────┴──────┴─────┘</pre> | `enabled_layouts horizontal` |
| **Vertical** | <pre>┌──────────────────┐<br>│                  │<br>├──────────────────┤<br>│                  │<br>├──────────────────┤<br>│                  │<br>└──────────────────┘</pre> | `enabled_layouts vertical` |

### Layout Controls

| Action | Shortcut | macOS |
|--------|----------|-------|
| Next layout | `Ctrl+Shift+L` | `Ctrl+Shift+L` |


## Copy/Paste

| Action | Shortcut | macOS |
|--------|----------|-------|
| Copy to clipboard | `Ctrl+Shift+C` | `Cmd+C` |
| Paste from clipboard | `Ctrl+Shift+V` | `Cmd+V` |

### Copy-on-Select and Right-Click Paste

Enable PuTTY-like behavior where selecting text automatically copies it, and right-click pastes:

```
# Copy selected text automatically to clipboard
copy_on_select yes

# Enable paste on right-click (disables right-click extend selection)
mouse_map right press ungrabbed paste_from_selection
```

Alternative: Copy to system clipboard instead of selection buffer:

```
copy_on_select clipboard
mouse_map right press ungrabbed paste_from_clipboard
```

Optional: Disable URL detection if it interferes with selection:

```
detect_urls no
allow_hyperlinks no
```

Note: On macOS, there's no distinction between clipboard and selection, so both methods work identically.

## Scrolling

| Action | Shortcut | macOS |
|--------|----------|-------|
| Line up | `Ctrl+Shift+Up` | `Cmd+Up` |
| Line down | `Ctrl+Shift+Down` | `Cmd+Down` |
| Page up | `Ctrl+Shift+PgUp` | `Cmd+PgUp` |
| Page down | `Ctrl+Shift+PgDn` | `Cmd+PgDn` |
| Top | `Ctrl+Shift+Home` | `Cmd+Home` |
| Bottom | `Ctrl+Shift+End` | `Cmd+End` |

### Scrollback & Output

| Action | Shortcut | macOS |
|--------|----------|-------|
| Browse scrollback in pager | `Ctrl+Shift+H` | `Ctrl+Shift+H` |
| Browse last command output | `Ctrl+Shift+G` | `Ctrl+Shift+G` |
| Search scrollback | `Ctrl+Shift+/` | `Ctrl+Shift+/` |

## OS Windows

| Action | Shortcut | macOS |
|--------|----------|-------|
| New OS window | `Ctrl+Shift+N` | `Cmd+N` |
| Close OS window | `Ctrl+Shift+W` | `Shift+Cmd+W` |
| Cycle OS windows | - | <code>Cmd+`</code> |
| Cycle backwards | - | <code>Cmd+Shift+`</code> |

## Font Size

| Action | Shortcut | macOS |
|--------|----------|-------|
| Increase font size | `Ctrl+Shift+=` | `Cmd+=` |
| Decrease font size | `Ctrl+Shift+-` | `Cmd+-` |
| Restore font size | `Ctrl+Shift+Backspace` | `Cmd+0` |

## Configuration

| Action | Shortcut | macOS |
|--------|----------|-------|
| Edit config file | `Ctrl+Shift+F2` | `Cmd+,` |
| Reload config | `Ctrl+Shift+F5` | `Ctrl+Cmd+,` |
| Show effective config | `Ctrl+Shift+F6` | `Opt+Cmd+,` |
| Show kitty docs | `Ctrl+Shift+F1` | - |

### Config File Location

```bash
~/.config/kitty/kitty.conf
```

### Common Settings

```
# Font
font_family      JetBrains Mono
font_size        12.0

# Mouse
copy_on_select   yes
strip_trailing_spaces smart

# Window
remember_window_size  yes
initial_window_width  640
initial_window_height 400
window_padding_width  5

# Colors
background_opacity 0.90
dynamic_background_opacity yes

# Scrollback
scrollback_lines 10000

# Layouts
enabled_layouts tall,stack,grid
```

### Launch Programs Examples

```
# Text editors
map cmd+shift+v launch --cwd=current vim
map cmd+shift+n launch --cwd=current nvim
map ctrl+shift+e launch --cwd=current code .

# File managers
map ctrl+shift+f launch --cwd=current ranger
map ctrl+shift+n launch --cwd=current nnn

# System monitors
map ctrl+shift+h launch --type=overlay htop
map ctrl+shift+b launch --type=overlay btop
```

Launch options:
- `--cwd=current` - starts in the same directory as current window
- `--type=overlay` - floating window on top
- `--type=tab` - new tab instead of split
- `--hold` - keeps window open after command exits
- `--title` - sets window title

## Themes

Browse and apply themes interactively:
```bash
kitten themes
```

![kitten themes](images/kitten-themes.png)

Or apply a specific theme directly:
```bash
kitten themes --reload-in=all <theme-name>
```

Popular themes:
- Catppuccin (Mocha, Latte, Frappe, Macchiato)
- Dracula
- Gruvbox
- Nord
- Tokyo Night
- Solarized

Theme files location: `~/.config/kitty/themes/`

## Shell Integration

Enables features like:
- Jump to previous/next prompts
- View last command output
- Click to open files from ls output
- Tab shows current directory and last command

## Kittens (Built-in Tools)

```bash
# View images in terminal
kitten icat image.png

# Choose theme interactively
kitten themes

# Hyperlinked grep (ripgrep with clickable results)
kitten hyperlinked_grep pattern

# Side-by-side diff for files and images
kitten diff file1.txt file2.txt

# SSH with automatic shell integration
kitten ssh hostname

# Transfer files over SSH
kitten transfer file.txt remote:/path/

# Clipboard operations (works over SSH)
kitten clipboard --get-clipboard
kitten clipboard file.txt

# File chooser dialog
kitten choose-files

# Font selector
kitten choose-fonts

# Edit file in overlay window
kitten edit-in-kitty file.txt

# Show keyboard codes for debugging
kitten show-key
```

### Broadcast Kitten

Type text simultaneously in multiple kitty windows. Useful for running the same commands across multiple servers or terminals.

```
# Setup keyboard shortcut in kitty.conf
map cmd+option+b launch --allow-remote-control kitty +kitten broadcast
```

Usage:
- Press the mapped key (Cmd+Option+B) to start broadcast mode
- Type in the broadcast window - text appears in all matched windows
- `Ctrl+Esc` - end broadcast session
- `Ctrl+Alt+Esc` - toggle hiding input (useful for passwords)

Common use cases:
- Running commands on multiple SSH sessions simultaneously
- Updating configuration across multiple terminals
- Testing commands in parallel environments

## Mouse Features

| Feature | Action |
|---------|--------|
| Open URL | Click URL |
| Open file path | `Ctrl+Shift+Right-click` on path |
| Select word | Double-click, drag to extend |
| Select line | Triple-click |
| Column selection | `Ctrl+Alt+Drag` |
| Select when app grabbed mouse | `Shift+Click` |
| Paste | Middle-click (on systems with primary selection) |

## Sessions

```bash
# Load session (session files go in ~/.config/kitty/)
kitty --session ~/.config/kitty/session.conf
```

### Session File Examples

```text
# 4-pane grid layout (2x2 equal)
# ┌─────────┬─────────┐
# │    1    │    2    │
# ├─────────┼─────────┤
# │    3    │    4    │
# └─────────┴─────────┘
layout grid
launch
launch
launch
launch
```

```text
# 4-pane tall layout (1 left + 3 stacked right)
# ┌─────────┬─────────┐
# │         │    2    │
# │         ├─────────┤
# │    1    │    3    │
# │         ├─────────┤
# │         │    4    │
# └─────────┴─────────┘
launch
launch --location=vsplit
launch --location=hsplit
launch --location=hsplit
```

## Remote Control

```bash
# Allow controlling kitty from the command line
allow_remote_control yes

# Control kitty from command line
kitty @ ls  # List windows
kitty @ set-colors --all background=black
kitty @ new-window
kitty @ close-window
```

## Marks (Text Highlighting)

Highlight and navigate through specific text patterns in terminal output - useful for logs and debugging.

```
# Create marks based on regex (use ctrl+alt on macOS or remap)
map ctrl+1 toggle_marker text 1 ERROR
map ctrl+2 toggle_marker text 2 WARNING
```

How it works:
- Press Ctrl+1 to highlight all lines containing "ERROR" (toggle on/off)
- Press Ctrl+2 to highlight all lines containing "WARNING" (toggle on/off)
- Multiple markers can be active simultaneously with different colors

## Resources

- Documentation: https://sw.kovidgoyal.net/kitty/
- Config reference: https://sw.kovidgoyal.net/kitty/conf/
- Actions reference: https://sw.kovidgoyal.net/kitty/actions/
- Kittens: https://sw.kovidgoyal.net/kitty/kittens_intro/
- GitHub: https://github.com/kovidgoyal/kitty
