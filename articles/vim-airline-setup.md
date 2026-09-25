# Installing and Theming vim-airline

`vim-airline` is a lightweight status/tabline for Vim, written in pure Vimscript with no external dependencies. It renders a rich status bar showing mode, file info, position, Git branch, and more, and it integrates with plugins like fugitive, ale, and coc. This guide covers installing it with Vim's native package system, applying a theme, and the common setup tweaks.

For a related Vim tweak, see [Vim White Spaces](articles/vim-white-spaces.md).

## Install with Vim's Native Package System (Vim 8+)

Vim 8 added native package loading: any plugin cloned into `~/.vim/pack/<name>/start/` is loaded automatically at startup — no plugin manager required. This is the cleanest way to install vim-airline.

```sh
# Create the package directory and clone airline into it
mkdir -p ~/.vim/pack/dist/start
git clone https://github.com/vim-airline/vim-airline.git \
  ~/.vim/pack/dist/start/vim-airline
```

Start Vim and the status bar appears immediately. To make it show even when only one window is open, enable it in `~/.vimrc`:

```vim
set laststatus=2
```

> `dist` above is just a package group name of your choosing — any name works. The required parts of the path are `pack/<group>/start/`, which Vim scans automatically on launch.

### Alternative: clone into ~/.vim directly

The upstream README also documents a manual install where the plugin files are copied into `~/.vim` itself. This works but predates native packages and mixes the plugin's files into your config directory:

```sh
mkdir -p ~/.vim
cd ~/.vim
git clone https://github.com/vim-airline/vim-airline.git
shopt -s dotglob && mv vim-airline/* . && rmdir vim-airline
```

Prefer the `pack/dist/start` method — it keeps each plugin self-contained and trivially removable (just delete its directory).

## Install a Theme

Themes ship in a separate repository. Clone it into the same package `start` directory so it loads automatically:

```sh
git clone https://github.com/vim-airline/vim-airline-themes \
  ~/.vim/pack/dist/start/vim-airline-themes
```

Then select a theme in `~/.vimrc`:

```vim
let g:airline_theme='<theme>'
```

Replace `<theme>` with a theme name, for example:

```vim
let g:airline_theme='dark'        " the default
let g:airline_theme='minimalist'
let g:airline_theme='badwolf'
let g:airline_theme='solarized'
```

List the themes available in your install, or preview one live:

```vim
:AirlineTheme <Tab>    " tab-complete to browse available theme names
:AirlineTheme badwolf  " switch theme in the current session to preview it
```

## Powerline Fonts (Optional but Recommended)

By default airline uses plain separators. The angled "powerline" separators and glyphs need a patched font installed and enabled:

```vim
let g:airline_powerline_fonts = 1
```

Install a patched font (for example from the [powerline/fonts](https://github.com/powerline/fonts) or [Nerd Fonts](https://www.nerdfonts.com/) projects) and set it as your terminal font. Without a patched font, leave this off or the separators render as garbled characters. If you use JetBrains Mono, a Nerd Font variant exists — see [JetBrains Mono Font](articles/jetbrains-mono-font.md).

## Enable the Tabline (Optional)

Show open buffers/tabs along the top:

```vim
let g:airline#extensions#tabline#enabled = 1
let g:airline#extensions#tabline#formatter = 'unique_tail'
```

## Minimal ~/.vimrc

A complete starter configuration:

```vim
set laststatus=2                              " always show the status line
let g:airline_theme = 'minimalist'            " pick a theme
let g:airline_powerline_fonts = 1             " use patched-font glyphs
let g:airline#extensions#tabline#enabled = 1  " show the buffer/tab line
```

## Updating and Removing

Because each plugin is its own Git checkout, maintenance is just Git and file operations:

```sh
# Update airline (and the themes) to the latest
cd ~/.vim/pack/dist/start/vim-airline && git pull
cd ~/.vim/pack/dist/start/vim-airline-themes && git pull

# Remove airline entirely
rm -rf ~/.vim/pack/dist/start/vim-airline
```

## Key Takeaways

- On Vim 8+, install airline by cloning into `~/.vim/pack/dist/start/` — no plugin manager needed.
- Add `set laststatus=2` so the status line shows even with a single window.
- Themes live in the separate `vim-airline-themes` repo; select one with `let g:airline_theme='<name>'`.
- Preview themes live with `:AirlineTheme <Tab>`.
- Set `g:airline_powerline_fonts = 1` only after installing and selecting a patched/Nerd Font.

## Quick Reference

```sh
# Install airline + themes (Vim 8+ native packages)
mkdir -p ~/.vim/pack/dist/start
git clone https://github.com/vim-airline/vim-airline.git \
  ~/.vim/pack/dist/start/vim-airline
git clone https://github.com/vim-airline/vim-airline-themes \
  ~/.vim/pack/dist/start/vim-airline-themes
```

```vim
" ~/.vimrc
set laststatus=2
let g:airline_theme = 'minimalist'
let g:airline_powerline_fonts = 1
let g:airline#extensions#tabline#enabled = 1
```

For related material, see [Vim White Spaces](articles/vim-white-spaces.md), [Vim Search and Replace](articles/vim-search-replace.md), and [JetBrains Mono Font](articles/jetbrains-mono-font.md).
