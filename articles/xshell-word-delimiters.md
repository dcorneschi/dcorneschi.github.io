# Xshell Word Delimiters for Double-Click Selection

In the [Xshell](https://www.netsarang.com/en/xshell/) terminal emulator, double-clicking a word selects it up to the nearest **delimiter** — a character that marks a word boundary. The default delimiter set treats most punctuation as a boundary, which means double-clicking a file path, URL, or flag like `--option=value` selects only a fragment instead of the whole token. Tuning the delimiter list fixes that. This guide explains the setting, where to find it, and how to adjust it.

For a comparable per-tool setting, see [PuTTY Default Settings](articles/putty-default-settings.md).

## What a Delimiter Does

When you double-click in the terminal, Xshell expands the selection left and right from the cursor until it hits a delimiter character. Everything between the two nearest delimiters becomes the selected "word".

- Characters **in** the delimiter set act as boundaries and are *not* included in the selection.
- Characters **not** in the set are treated as part of the word.

So the delimiter list is effectively "the characters that break a word apart." Removing a character from the list makes double-click treat it as part of a word instead of a boundary.

## The Default Delimiter Set

Xshell's default delimiters are the whitespace/backslash prefix plus this run of punctuation:

```text
\ :;~`!@#$%^&*()=+|[]{}'",<>?
```

Broken down, that set includes:

| Characters | Notes |
|------------|-------|
| `\` and space | backslash and whitespace |
| `:` `;` | colon, semicolon |
| `~` `` ` `` | tilde, backtick |
| `!` `@` `#` `$` `%` `^` `&` `*` | shell/metacharacter punctuation |
| `(` `)` `[` `]` `{` `}` | brackets and braces |
| `=` `+` `\|` | equals, plus, pipe |
| `'` `"` `,` `<` `>` `?` | quotes, comma, angle brackets, question mark |

Notably **absent** from the defaults are `/`, `.`, `-`, and `_`. Because those are *not* delimiters, double-clicking already selects across them — which is why full paths (`/var/log/messages`), dotted names (`file.tar.gz`), and hyphenated/underscored identifiers usually select as one unit out of the box.

## Where to Change It

The delimiters live in the **Mouse** settings, per NetSarang's documentation, and can be set globally or per session:

- **Global default:** `Tools` → `Options` → `Mouse` — sets the delimiters for new sessions.
- **Per session:** open the session's **Properties** → `Terminal` → `Mouse` (category names vary slightly across Xshell versions).

Look for the field labeled for word-selection delimiters, edit the character list, and click OK. The change takes effect for new double-click selections; existing sessions may need to be reopened for a global change to apply.

> Menu paths differ between Xshell versions (6, 7, 8). If you don't see it under `Mouse`, search the session Properties tree for the mouse/selection category. The behavior — a single editable list of delimiter characters — is the same across versions.

## Tuning Examples

**Select whole URLs and paths including query strings.** By default `:`, `?`, `=`, `&`, and `#` are delimiters, so a URL like `https://host/p?a=1&b=2#frag` selects only in pieces. Remove those characters from the delimiter list to grab the entire URL in one double-click.

**Keep shell flags intact.** `=` is a delimiter by default, so double-clicking `--name=value` stops at the `=`. Remove `=` from the set to select the whole `key=value`.

**Make double-click stop at dots.** If you *want* `file.tar.gz` to select just `gz`, do the opposite — **add** `.` to the delimiter list so the dot becomes a boundary.

## Selection Shortcuts

Delimiters only affect double-click. Xshell also supports:

- **Double-click** — select a word (bounded by delimiters).
- **Triple-click** — select the entire line, regardless of delimiters. Useful when the delimiter set is fighting you.
- **Click + drag** — free-form selection of any range.

## Key Takeaways

- Delimiters are the characters that break words apart for **double-click** selection; they are not included in the selection.
- Xshell's default set is `` \ :;~`!@#$%^&*()=+|[]{}'",<>? `` — note `/`, `.`, `-`, `_` are **not** delimiters, so paths and identifiers select whole.
- Configure them under `Tools` → `Options` → `Mouse` (global) or session **Properties** → `Mouse` (per session); paths vary by version.
- **Remove** a character to include it in words (e.g. drop `=` to grab `key=value`); **add** one to make it a boundary.
- Triple-click selects the whole line and ignores delimiters entirely.

For related material, see [PuTTY Default Settings](articles/putty-default-settings.md).

## Links

- NetSarang: [Xshell Mouse settings (word-selection delimiters)](https://netsarang.atlassian.net/wiki/spaces/EN8/pages/2237305380/Mouse+Setting)
