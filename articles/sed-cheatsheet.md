# sed Cheatsheet

sed (stream editor) performs text transformations on input streams or files. It processes text line by line, applying editing commands without opening an interactive editor.

## Syntax

```bash
sed [options] 'command' file
sed [options] -e 'command1' -e 'command2' file
sed [options] -f script.sed file
command | sed 'command'
```

## Common Options

| Option | Description |
|--------|-------------|
| `-i` | Edit file in place |
| `-i.bak` | Edit in place, create backup with `.bak` extension |
| `-n` | Suppress automatic printing (use with `p`) |
| `-e` | Add multiple commands |
| `-f file` | Read commands from file |
| `-r` / `-E` | Extended regex (ERE) |

> **Note:** On macOS/BSD, `-i` requires an argument: `sed -i '' 's/old/new/' file`

## Substitution (s)

### Basic Syntax

```bash
sed 's/old/new/' file          # Replace first occurrence per line
sed 's/old/new/g' file         # Replace all occurrences per line
sed 's/old/new/2' file         # Replace 2nd occurrence per line
sed 's/old/new/gi' file        # Replace all, case-insensitive
sed 's/old/new/p' file         # Print lines where substitution was made
sed -n 's/old/new/p' file      # Only print lines with substitutions
```

### In-Place Editing

```bash
sed -i 's/old/new/g' file              # Linux
sed -i '' 's/old/new/g' file           # macOS/BSD
sed -i.bak 's/old/new/g' file          # With backup
```

### Delimiters

Any character can be used as delimiter:

```bash
sed 's|/usr/local|/opt|g' file         # Use | for paths
sed 's#http://#https://#g' file        # Use # for URLs
sed 's@old@new@g' file                 # Use @
```

### Capture Groups

```bash
# Swap two words
sed 's/\(word1\) \(word2\)/\2 \1/' file

# Extended regex (no backslash needed)
sed -E 's/(word1) (word2)/\2 \1/' file

# Add quotes around a word
sed 's/\(pattern\)/"\1"/' file

# Extract part of a line
sed -E 's/.*name="([^"]+)".*/\1/' file
```

### Special Replacement Characters

| Character | Description |
|-----------|-------------|
| `&` | Matched text |
| `\1, \2...` | Capture group back-references |
| `\n` | Newline (in replacement) |
| `\l` | Lowercase next character |
| `\u` | Uppercase next character |
| `\L` | Lowercase until `\E` |
| `\U` | Uppercase until `\E` |

```bash
# Surround match with brackets
sed 's/[0-9]*/[&]/' file

# Uppercase first letter of each word
sed 's/\b\(.\)/\u\1/g' file
```

## Deletion (d)

```bash
sed '3d' file                  # Delete line 3
sed '2,5d' file                # Delete lines 2-5
sed '$d' file                  # Delete last line
sed '/pattern/d' file          # Delete lines matching pattern
sed '/^$/d' file               # Delete empty lines
sed '/^#/d' file               # Delete comment lines
sed '1d' file                  # Delete first line (header)
sed '/^$/d;/^#/d' file         # Delete empty and comment lines
```

## Insertion and Appending

```bash
# Insert before line 3
sed '3i\New line text' file

# Append after line 3
sed '3a\New line text' file

# Insert before matching line
sed '/pattern/i\New line text' file

# Append after matching line
sed '/pattern/a\New line text' file

# Insert at beginning of file
sed '1i\First line' file

# Append at end of file
sed '$a\Last line' file
```

## Change (c)

```bash
# Replace entire line 3
sed '3c\Replacement line' file

# Replace lines matching pattern
sed '/pattern/c\Replacement line' file
```

## Print (p)

```bash
# Print specific lines
sed -n '5p' file               # Print line 5
sed -n '3,7p' file             # Print lines 3-7
sed -n '$p' file               # Print last line
sed -n '/pattern/p' file       # Print matching lines
sed -n '1,/pattern/p' file     # Print from start to pattern
sed -n '/start/,/end/p' file   # Print between patterns
```

## Line Addressing

### By Line Number

```bash
sed '5s/old/new/' file         # Substitute only on line 5
sed '1,10s/old/new/' file      # Lines 1-10
sed '5,$s/old/new/' file       # Line 5 to end
sed '1~2s/old/new/' file       # Odd lines (1, 3, 5...)
sed '0~2s/old/new/' file       # Even lines (2, 4, 6...)
```

### By Pattern

```bash
sed '/pattern/s/old/new/' file             # Lines matching pattern
sed '/start/,/end/s/old/new/' file         # Between two patterns
sed '/pattern/!s/old/new/' file            # Lines NOT matching pattern
```

## Multiple Commands

```bash
# Using -e
sed -e 's/foo/bar/' -e 's/baz/qux/' file

# Using semicolons
sed 's/foo/bar/; s/baz/qux/' file

# Using newlines (in script files)
sed '
s/foo/bar/
s/baz/qux/
/pattern/d
' file
```

## Hold Space and Pattern Space

```bash
# Swap pattern and hold space
sed 'x' file

# Copy pattern to hold space
sed 'h' file

# Append pattern to hold space
sed 'H' file

# Get hold space to pattern space
sed 'g' file

# Append hold space to pattern space
sed 'G' file

# Double space a file (append newline after each line)
sed 'G' file

# Reverse line order
sed -n '1!G;h;$p' file
```

## Practical Examples

### File Editing

```bash
# Remove trailing whitespace
sed 's/[[:space:]]*$//' file

# Remove leading whitespace
sed 's/^[[:space:]]*//' file

# Remove both leading and trailing whitespace
sed 's/^[[:space:]]*//;s/[[:space:]]*$//' file

# Remove blank lines
sed '/^$/d' file

# Remove lines containing pattern
sed '/DEBUG/d' logfile

# Add line numbers
sed = file | sed 'N;s/\n/\t/'
```

### Configuration Files

```bash
# Uncomment a line
sed -i 's/^#\(PermitRootLogin\)/\1/' /etc/ssh/sshd_config

# Comment a line
sed -i 's/^\(PermitRootLogin\)/#\1/' /etc/ssh/sshd_config

# Change a config value
sed -i 's/^MaxRetries.*/MaxRetries 5/' config.conf

# Add a line after a match
sed -i '/\[section\]/a\new_key = new_value' config.ini

# Replace between markers
sed -i '/BEGIN/,/END/c\New content between markers' file
```

### Text Processing

```bash
# Extract email addresses
sed -n 's/.*\([a-zA-Z0-9._%+-]*@[a-zA-Z0-9.-]*\.[a-zA-Z]*\).*/\1/p' file

# Convert DOS line endings to Unix
sed -i 's/\r$//' file

# Convert Unix line endings to DOS
sed -i 's/$/\r/' file

# Print lines between line numbers
sed -n '10,20p' file

# Print nth line
sed -n '5p' file

# Replace newlines with spaces (join lines)
sed ':a;N;$!ba;s/\n/ /g' file

# Insert blank line every 5 lines
sed '0~5G' file
```

### Search and Replace Patterns

```bash
# Replace only if line contains another pattern
sed '/error/s/status: [0-9]*/status: 0/' file

# Replace in specific line range
sed '10,20s/old/new/g' file

# Replace everything after a match
sed 's/keyword.*/keyword: new_value/' file

# Replace everything before a match
sed 's/.*keyword/new_prefix keyword/' file

# Delete from pattern to end of file
sed '/pattern/,$d' file

# Delete from start to pattern
sed '1,/pattern/d' file
```

### Multi-Line Operations

```bash
# Join continuation lines (ending with \)
sed -e ':a' -e '/\\$/N; s/\\\n//; ta' file

# Delete a block of lines between patterns
sed '/START/,/END/d' file

# Replace across lines
sed 'N;s/\n//' file
```

## Regex Reference

| Pattern | Matches |
|---------|---------|
| `.` | Any single character |
| `*` | Zero or more of preceding |
| `+` | One or more (ERE, use `-E`) |
| `?` | Zero or one (ERE, use `-E`) |
| `^` | Start of line |
| `$` | End of line |
| `[abc]` | Character class |
| `[^abc]` | Negated class |
| `\(group\)` | Capture group (BRE) |
| `(group)` | Capture group (ERE, use `-E`) |
| `\{n,m\}` | Repetition (BRE) |
| `{n,m}` | Repetition (ERE) |
| `\b` | Word boundary |

### POSIX Character Classes

| Class | Matches |
|-------|---------|
| `[[:alpha:]]` | Letters |
| `[[:digit:]]` | Digits |
| `[[:alnum:]]` | Letters and digits |
| `[[:space:]]` | Whitespace |
| `[[:upper:]]` | Uppercase |
| `[[:lower:]]` | Lowercase |
| `[[:punct:]]` | Punctuation |

## One-Liner Reference

| Command | Description |
|---------|-------------|
| `sed -n '5p'` | Print line 5 |
| `sed -n '$p'` | Print last line |
| `sed '1d'` | Delete first line |
| `sed '$d'` | Delete last line |
| `sed '/^$/d'` | Delete blank lines |
| `sed 's/old/new/g'` | Replace all occurrences |
| `sed -n '/pattern/p'` | Print matching lines (grep) |
| `sed '/pattern/d'` | Delete matching lines (inverse grep) |
| `sed 'G'` | Double space |
| `sed -n '1!G;h;$p'` | Reverse lines (tac) |
| `sed '10q'` | Print first 10 lines (head) |
| `sed -e :a -e '$q;N;11,$D;ba'` | Print last 10 lines (tail) |
| `sed = file \| sed 'N;s/\n/\t/'` | Number lines (cat -n) |
| `sed 's/^/    /'` | Indent 4 spaces |
| `sed 's/.\{4\}//'` | Remove first 4 characters |

## GNU sed vs BSD sed (macOS)

| Feature | GNU sed | BSD sed (macOS) |
|---------|---------|-----------------|
| In-place edit | `sed -i 's/...'` | `sed -i '' 's/...'` |
| Extended regex | `sed -r` or `sed -E` | `sed -E` |
| `\t` in replacement | Supported | Use literal tab |
| `\n` in replacement | Supported | Limited support |
| Case conversion `\u \l` | Supported | Not supported |

## Advanced Pattern Matching

### Match Line After Pattern

```bash
sed '/pattern/{n;s/old/new/}' file     # Substitute on line after match
sed '/pattern/{n;d}' file              # Delete line after match
sed '/pattern/{n;c\new_line_content
}' file                                # Replace line after match entirely
```

### Match Line Before Pattern

```bash
sed -n '/pattern/{x;p;};h' file        # Print line before match
```

### Relative Range Addressing

```bash
sed '10,+5s/old/new/' file             # Line 10 plus next 5 lines
```

### Negation

```bash
sed '/pattern/!d' file                 # Delete lines NOT matching (keep only matches)
sed -n '1,5!p' file                    # Print all lines except 1-5
```

### Grouping Commands

```bash
sed '/pattern/{s/old/new/; s/foo/bar/}' file
```

### Print Line Numbers

```bash
sed -n '=' file                        # Print line numbers only
sed '=' file                           # Print line numbers with content
```

### Remove Duplicate Consecutive Lines

```bash
sed '$!N; /^\(.*\)\n\1$/!P; D' file
```

### Replace Newlines with Commas

```bash
sed ':a;N;$!ba;s/\n/,/g' file
```

### Double Space (simple form)

```bash
sed G file
```

## Real-World Sysadmin Examples

```bash
# Delete a specific line from known_hosts
sed -i 14d ~/.ssh/known_hosts

# Replace only on first 30 lines
sed -i '1,30s/nagios/nagiosadmin/g' /usr/local/nagios/etc/nrpe.cfg

# Change string only on lines matching another pattern
sed -i '/nrpe/s/nagiosadmin/nagios/' /usr/local/nagios/etc/nrpe.cfg

# Add prefix to beginning of each line
sed -i 's/^/pvcreate \/dev\//g' emc_devices.lst

# Enable all repos in a file
sed -i "s/enabled=0/enabled=1/g" /etc/yum.repos.d/epel.repo

# Replace value for a setting (keep the key)
sed -i.bak '/^GRUB_DEFAULT=/s/=.*/=1/' /etc/default/grub
sed -i.bak 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT=1/' /etc/default/grub

# Append argument to end of a specific line
sed -e '/check_netstat/s/$/ \$ARG3\$/' /usr/local/nagios/etc/nrpe.cfg

# Backup and append IP to allowed_hosts
sed -i.bak '/^allowed_hosts=/s/$/,192.168.1.27/' /usr/local/nagios/etc/nrpe.cfg

# Toggle a boolean setting
sed -i 's/^dont_blame_nrpe=0/dont_blame_nrpe=1/g' /usr/local/nagios/etc/nrpe.cfg

# Delete all comment lines and blank lines
sed -i '/^#/d; /^$/d' /usr/local/nagios/etc/nrpe.cfg

# Append a line at end of file
sed -i '$a\My final line' /etc/fstab

# Remove trailing spaces
sed -i 's/[[:space:]]*$//' aws-auth-backup.yaml

# Print first 5 lines
sed -n '1,5p' file

# Print last line
sed -n '$p' file

# Print all lines except first 5
sed -n '1,5!p' file

# Print lines matching pattern
sed -n '/repo/p' anaconda-ks.cfg
```

## Append Command Deep Dive

### Syntax Breakdown

```text
sed -i '1a text'
│   │  │ │ │
│   │  │ │ └── Text to append
│   │  │ └──── 'a' = append command
│   │  └────── '1' = line number (after line 1)
│   └───────── '-i' = edit file in-place
└─────────── sed = stream editor
```

### Insert vs Append

```bash
sed -i '1i text'     # Insert BEFORE line 1
sed -i '1a text'     # Append AFTER line 1
```

### Modern vs Traditional Syntax

```bash
# Modern (GNU sed only, single line)
sed -i '1a New line' file.txt

# Traditional (POSIX, works everywhere)
sed -i '1a\
New line' file.txt

# Just 'a' alone = error
sed -i '1a' file.txt
# Error: expected \ after `a', `c' or `i'
```

### Syntax Compatibility

| Syntax | GNU sed (Linux) | POSIX sed | Multi-line |
|--------|-----------------|-----------|------------|
| `'1a text'` | Yes | No | No |
| `'1a\'` + text on next line | Yes | Yes | Yes |
| `'1a'` alone | Error | Error | Error |

### Multi-Line Append

```bash
# Traditional backslash continuation (works everywhere)
sed -i '/\[Service\]/a\
LimitNOFILE=infinity\
LimitNPROC=infinity\
LimitCORE=infinity' override.conf

# GNU sed with \n (Linux only)
sed -i '/\[Service\]/a LimitNOFILE=infinity\nLimitNPROC=infinity' file

# Using here-doc with /r
sed -i '/\[Service\]/r /dev/stdin' file << EOF
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
EOF
```

### Conditional Append

```bash
# Only append if line doesn't already exist
grep -q "LimitNOFILE" file || sed -i '/\[Service\]/a LimitNOFILE=infinity' file

# Append only if pattern exists but target line doesn't
sed -i '/\[Service\]/{ /LimitNOFILE/!a LimitNOFILE=infinity
}' service.file
```

### Before/After Example

```bash
# Original file:
# Line 1
# Line 2
# Line 3

sed -i '1a New Line' file.txt

# Result:
# Line 1
# New Line    ← inserted after line 1
# Line 2
# Line 3
```

### systemd Service Configuration

```bash
# Add LimitNOFILE after [Service] in containerd override
mkdir -p /etc/systemd/system/containerd.service.d
echo "[Service]" > /etc/systemd/system/containerd.service.d/override.conf
sed -i '/\[Service\]/a LimitNOFILE=infinity' /etc/systemd/system/containerd.service.d/override.conf

# Add multiple limits
sed -i '/\[Service\]/a\
LimitNOFILE=infinity\
LimitNPROC=infinity\
LimitCORE=infinity' /etc/systemd/system/containerd.service.d/override.conf

# Add after specific directives
sed -i '/^ExecStart/a ExecStartPost=/bin/echo "Started"' service.file

# Add config after section headers
sed -i '/\[database\]/a connection_timeout=30' config.ini
```

### Log and File Annotations

```bash
# Add timestamp after first line
sed -i "1a # Modified on $(date)" logfile.txt

# Add separator after ERROR lines
sed -i '/ERROR/a ----------------------------------------' log.txt
```

## Common Errors and Fixes

### Missing Text After Append

```bash
# Wrong (incomplete)
sed -i '1a ' file.txt              # Appends empty line (probably unintended)

# Correct
sed -i '1a Some text' file.txt
```

### Unescaped Special Characters

```bash
# Wrong (literal brackets won't match)
sed -i '/[Service]/a text' file

# Correct (escape brackets)
sed -i '/\[Service\]/a text' file
```

### Quotes and Backslashes

```bash
# Escape special characters in appended text
sed -i '1a Line with "quotes"' file.txt
sed -i '1a Line with \$variables' file.txt
```

## Tips

- Always test with `sed 's/old/new/g' file` before using `-i`
- Use `sed -i.bak` to create a backup before in-place edits
- Use `-E` for extended regex to avoid excessive backslash escaping
- Use different delimiters (`|`, `#`, `@`) when working with paths or URLs
- Chain multiple commands with `-e` or semicolons
- Combine with `find` for bulk edits: `find . -name "*.conf" -exec sed -i 's/old/new/g' {} +`
