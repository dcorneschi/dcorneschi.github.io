# tr Cheatsheet

`tr` (translate) reads from **standard input**, transforms or deletes characters, and writes to **standard output**. It works only on single characters — it cannot match strings or patterns — which makes it fast and predictable for tasks like changing case, squeezing whitespace, or stripping unwanted bytes. Because it only reads stdin, you pipe into it or redirect a file with `<`; it does not take a filename argument.

For related text tools, see the [sed Cheatsheet](articles/sed-cheatsheet.md), the [awk Cheatsheet](articles/awk-cheatsheet.md), and the [cut Cheatsheet](articles/cut-cheatsheet.md).

## Syntax

```text
tr [OPTIONS] SET1 [SET2]
```

`tr` maps each character in `SET1` to the character at the same position in `SET2`. With `-d` it deletes `SET1`; with `-s` it squeezes repeats.

## Core Options

| Option | Meaning |
|--------|---------|
| `-d` | Delete characters in `SET1` |
| `-s` | Squeeze repeated characters in `SET1` into one |
| `-c` | Complement `SET1` (match everything *not* in it) |
| `-t` | Truncate `SET1` to the length of `SET2` |

## Translate Characters

```sh
tr 'a-z' 'A-Z' < file        # lowercase to uppercase
tr 'A-Z' 'a-z' < file        # uppercase to lowercase
echo 'hello' | tr 'l' 'L'    # hello -> heLLo
tr 'abc' 'xyz' < file        # a->x, b->y, c->z
```

If `SET2` is shorter than `SET1`, its last character is repeated to fill the gap (unless `-t` is used).

## Squeeze Repeats (-s)

Collapse runs of a character into a single instance:

```sh
tr -s ' ' < file                       # collapse repeated spaces
tr -s '[[:space:]]' '\n'               # transform whitespace runs into single newlines
echo 'aaabbbccc' | tr -s 'abc'         # -> abc
```

The `tr -s '[[:space:]]' '\n'` idiom turns any run of spaces, tabs, or newlines into a single newline — handy for putting each word on its own line.

## Delete Characters (-d)

```sh
tr -d '\r' < dosfile > unixfile        # strip carriage returns (CRLF -> LF)
tr -d '[:digit:]' < file               # remove all digits
echo 'h e l l o' | tr -d ' '           # remove spaces -> hello
tr -d '\n' < file                      # join all lines into one
```

## Complement (-c)

`-c` inverts `SET1` so the operation applies to every character *not* listed. Combined with `-d` or `-s` it's a powerful whitelist:

```sh
tr -cd '[:alnum:]' < file              # keep only letters and digits, delete the rest
tr -cs '[:alnum:]' '\n' < file         # split on any non-alphanumeric run -> one token per line
echo 'abc123!@#' | tr -cd '[:alpha:]'  # -> abc
```

## Character Classes

Use POSIX classes instead of spelling out ranges:

| Class | Matches |
|-------|---------|
| `[:alpha:]` | letters |
| `[:digit:]` | digits |
| `[:alnum:]` | letters and digits |
| `[:space:]` | whitespace (space, tab, newline, …) |
| `[:blank:]` | space and tab |
| `[:upper:]` | uppercase letters |
| `[:lower:]` | lowercase letters |
| `[:punct:]` | punctuation |
| `[:cntrl:]` | control characters |
| `[:print:]` | printable characters |

```sh
tr '[:upper:]' '[:lower:]' < file      # lowercase using classes
tr -d '[:punct:]' < file               # strip punctuation
```

## Escape Sequences and Ranges

`tr` understands C-style escapes and range notation in a set:

| Notation | Meaning |
|----------|---------|
| `\n` | newline |
| `\t` | tab |
| `\r` | carriage return |
| `\\` | backslash |
| `\0` | null |
| `a-z` | range from `a` to `z` |
| `[a*n]` | character `a` repeated `n` times (pad `SET2`) |

## Common One-Liners

```sh
# Put each word on its own line
tr -s '[[:space:]]' '\n' < file

# ROT13 (self-inverse cipher)
echo 'Hello' | tr 'A-Za-z' 'N-ZA-Mn-za-m'

# Count words by splitting on non-alphanumerics, then sort/uniq
tr -cs '[:alnum:]' '\n' < file | sort | uniq -c | sort -nr

# Convert Windows line endings to Unix
tr -d '\r' < dos.txt > unix.txt

# Remove all non-printable characters
tr -cd '[:print:]\n' < file

# Generate a random password from /dev/urandom
tr -dc '[:alnum:]' < /dev/urandom | head -c 16; echo
```

## Notes and Gotchas

- **stdin only.** `tr` has no filename argument; use `tr ... < file` or a pipe. `tr ... file` will not work.
- **Single characters, not strings.** `tr` cannot translate a multi-character string to another string — for that use [`sed`](articles/sed-cheatsheet.md) or [`awk`](articles/awk-cheatsheet.md).
- **`SET2` shorter than `SET1`** repeats `SET2`'s last character to match, unless you pass `-t` to truncate `SET1` instead.
- **Quote your sets** so the shell doesn't interpret `*`, `[`, or spaces before `tr` sees them.

## Quick Reference

```sh
tr 'a-z' 'A-Z' < file            # translate (lower -> upper)
tr -d '\r' < file                # delete characters
tr -s ' ' < file                 # squeeze repeats
tr -s '[[:space:]]' '\n' < file  # whitespace runs -> single newlines
tr -cd '[:alnum:]' < file        # keep only alphanumerics (complement + delete)
tr -cs '[:alnum:]' '\n' < file   # one token per line
```

For related material, see the [sed Cheatsheet](articles/sed-cheatsheet.md), the [awk Cheatsheet](articles/awk-cheatsheet.md), and the [cut Cheatsheet](articles/cut-cheatsheet.md).
