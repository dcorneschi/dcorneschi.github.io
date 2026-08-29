# Kiro CLI Cheatsheet

[Kiro CLI](https://kiro.dev/docs/cli/) (`kiro-cli`) brings Kiro's AI-assisted development to the terminal — an interactive chat agent with custom agents, MCP integration, steering, and hooks, built on the same agent harness as the Kiro IDE. This covers installation, authentication, starting and managing chat sessions, and the in-session slash commands.

> Kiro CLI is available on **macOS and Linux**. The binary is `kiro-cli`. This cheatsheet reflects the docs as of late 2025 — the tool evolves quickly, so run `kiro-cli --help` (and `kiro-cli <command> --help`) or check [kiro.dev/docs/cli](https://kiro.dev/docs/cli/) for the authoritative, version-specific command and flag list.

## Installation

```bash
# macOS / Linux
curl -fsSL https://cli.kiro.dev/install | bash

# Verify
kiro-cli --version
kiro-cli --help
```

## Authentication

Kiro CLI signs in through GitHub, Google, AWS Builder ID, or AWS IAM Identity Center.

```bash
kiro-cli login       # sign in (opens a browser to complete auth); plain `kiro-cli` also prompts
kiro-cli logout      # sign out
```

The first `kiro-cli` run prompts you to press Enter and finish sign-in in the browser, then returns you to the terminal authenticated.

## Starting a Chat Session

The core of the CLI is interactive chat.

```bash
kiro-cli                      # start an interactive chat in the current directory
kiro-cli --agent myagent      # start with a specific custom agent
kiro-cli chat --resume        # resume the previous conversation for this directory
```

Kiro keeps conversation history **per directory** — start `kiro-cli` in a folder you've used before and you can resume where you left off with `chat --resume`.

## In-Session Slash Commands

Inside a chat session, `/`-prefixed commands perform actions without leaving the conversation. Type `/` to see the list; common ones:

| Command | Action |
|---------|--------|
| `/editor` | Compose a multi-line prompt in your `$EDITOR` (defaults to vi); sent on save/close |
| `/reply` | Open the editor with the last assistant message quoted, for a multi-line reply |
| `/save [path]` | Save the current conversation to a JSON file (`-f`/`--force` to overwrite) |
| `/load [path]` | Load a conversation from a previously saved JSON file (replaces the current one) |
| `/settings` | Open in-session settings; or jump to a subcommand: `theme`, `keybindings`, `terminal`, `display`, `history` |
| `/help` | Show available slash commands |
| `/quit` | Exit the chat session |

```text
/editor
/reply
/save ./my-project-conversation -f
/load ./my-project-conversation.json
/settings theme
```

> `/save` and `/load` work independently of the directory where the conversation was created, and `/load` **replaces** your current conversation. You can't use `~` for your home directory in these paths.

### Multi-line input

- `/editor` — compose in your external editor.
- `Ctrl + J` — insert a newline directly in the prompt for a multi-line message.

## Key Concepts

Kiro CLI shares its configuration model (`.kiro/`) with the IDE, so these are portable across surfaces:

| Feature | What it does | Docs |
|---------|--------------|------|
| **Custom agents** | Task-specific agents with defined tools, context, and prompts (`--agent <name>`) | [custom-agents](https://kiro.dev/docs/cli/custom-agents/) |
| **MCP** | Connect external tools/data via Model Context Protocol servers | [mcp](https://kiro.dev/docs/cli/mcp/) |
| **Steering** | Guide the agent with your standards/preferences (`.kiro/steering/`) | [steering](https://kiro.dev/docs/cli/steering/) |
| **Hooks** | Automate pre/post actions around agent/tool events | [hooks](https://kiro.dev/docs/cli/hooks/) |
| **Autocomplete** | Context-aware command completion | [autocomplete](https://kiro.dev/docs/cli/autocomplete/) |

## Settings

- In-session: `/settings` (and subcommands `theme`, `keybindings`, `terminal`, `display`, `history`).
- The full settings reference (telemetry, chat behavior, key bindings, feature toggles) is documented at [kiro.dev/docs/reference/settings](https://kiro.dev/docs/reference/settings/).

## Quick Reference

| Task | Command |
|------|---------|
| Install | `curl -fsSL https://cli.kiro.dev/install \| bash` |
| Start chat | `kiro-cli` |
| Start with an agent | `kiro-cli --agent <name>` |
| Resume this dir's chat | `kiro-cli chat --resume` |
| Sign in / out | `kiro-cli login` / `kiro-cli logout` |
| Help | `kiro-cli --help` |
| Multi-line prompt | `/editor` (or `Ctrl+J`) |
| Save / load conversation | `/save <path>` / `/load <path>` |
| In-session settings | `/settings` |
| Exit chat | `/quit` |

## Resources

- CLI docs: [kiro.dev/docs/cli](https://kiro.dev/docs/cli/)
- CLI commands reference: [kiro.dev/docs/reference/cli-commands](https://kiro.dev/docs/reference/cli-commands/)
- Slash commands: [kiro.dev/docs/reference/slash-commands](https://kiro.dev/docs/reference/slash-commands/)
- Settings: [kiro.dev/docs/reference/settings](https://kiro.dev/docs/reference/settings/)

Verify commands and flags against the official docs (or `kiro-cli --help`), as the CLI changes frequently.

## Related

- [VS Code Git Actions and Git CLI Equivalents](articles/vscode-git-cli-equivalents.md) — another editor/CLI mapping for a common dev tool
