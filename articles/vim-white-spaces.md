## Configuring vim to show white spaces

### Customizing Symbols

```sh
vi ~/.vimrc
```

```sh
set listchars=tab:>-,eol:$,trail:~,extends:>,precedes:<
```

* tab: `>-` - Shows tabs (by default `^I`).
* eol `$`: Represents the end of a line.
* trail: `~` - Indicates trailing spaces.
* extends: `>` - Used for visual indication in line wrapping modes.
* precedes: `<` - Indicates that text extends to the left off-screen in line wrapping modes.

### Activate the mode in vim

Activate hidden characters in vim with `:set list` command.
