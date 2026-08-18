# Display Tabs and Whitespace in Files

Methods for revealing invisible characters — tabs, spaces, trailing whitespace, and non-printable characters — using common Linux tools and editors.

## cat

```bash
# Show tabs as ^I
cat -A file.txt

# Show tabs as ^I (without end-of-line markers)
cat -T file.txt

# Show end of lines as $ and tabs as ^I
cat -ET file.txt

# Show all non-printing characters
cat -v file.txt

# Combination: tabs, EOL, and non-printing
cat -vET file.txt
```

### cat Flags

| Flag | Shows |
|------|-------|
| `-A` | All (`-vET` combined) — tabs as `^I`, EOL as `$` |
| `-T` | Tabs as `^I` |
| `-E` | End of line as `$` |
| `-v` | Non-printing characters |

## sed

```bash
# Show tabs as literal [TAB]
sed -n 'l' file.txt

# Print only lines containing tabs
sed -n '/\t/p' file.txt

# Replace tabs with visible marker
sed 's/\t/→/g' file.txt

# Replace tabs with [TAB] text
sed 's/\t/[TAB]/g' file.txt

# Show trailing spaces
sed 's/ $/·/' file.txt
```

## awk

```bash
# Show tab positions
awk '{gsub(/\t/, "→"); print}' file.txt

# Show tabs and count them
awk '{n=gsub(/\t/, "→"); print n, $0}' file.txt

# Print only lines containing tabs
awk '/\t/' file.txt
```

## grep

```bash
# Find lines containing tabs
grep -P '\t' file.txt
grep -nP "\t" file.txt

# Alternative tab syntax (no -P needed)
grep "$(printf '\t')" file.txt
grep -n $'\t' file.txt

# Highlight tabs in output
grep -P --color=always '\t' file.txt

# Find files containing tabs
grep -rlP '\t' /path/to/dir/

# Count lines with tabs
grep -cP '\t' file.txt

# Find trailing whitespace
grep -n ' $' file.txt
grep -nP '\s+$' file.txt
```

## hexdump / od

```bash
# Show hex values (tab = 09, space = 20)
hexdump -C file.txt | head

# Show with character representation
od -c file.txt | head

# Just show unique characters in a file
od -c file.txt | awk '{for(i=2;i<=NF;i++) print $i}' | sort -u
```

## expand / unexpand

```bash
# Convert tabs to spaces (default: 8)
expand file.txt

# Convert tabs to 4 spaces
expand -t 4 file.txt

# Convert spaces to tabs
unexpand file.txt

# Convert leading spaces to tabs (4 spaces per tab)
unexpand -t 4 --first-only file.txt
```

## vim

```bash
# Show whitespace characters
:set list

# Hide whitespace characters
:set nolist

# Customize display characters
:set listchars=tab:→\ ,trail:·,extends:»,precedes:«,nbsp:␣

# Show tabs as >--- and trailing spaces as ·
:set listchars=tab:>-,trail:·

# Toggle list mode
:set list!
```

### Permanent vim Configuration

Add to `~/.vimrc`:

```vim
set list
set listchars=tab:→\ ,trail:·,extends:»,precedes:«,nbsp:␣
```

### Temporary Tab Highlighting

For quick one-off use, search for tabs to highlight them:

```vim
/\t
```

To remove the highlighting:

```vim
:noh
```

## nano

```bash
# Open file (use Alt+P inside nano to toggle whitespace display)
nano file.txt
```

In `~/.nanorc`:

```bash
set whitespace "→ "
set tabsize 4
```

## less

```bash
# Show control characters (tabs visible)
less -U file.txt

# Show with raw control characters
less -r file.txt
```

## file Command

```bash
# Detect line endings and encoding
file file.txt
```

Output might show: `ASCII text`, `ASCII text, with CRLF line terminators`, `UTF-8 Unicode text`.

## One-Liners

### Find Files with Tabs

```bash
# Find all files with tabs in a directory
find . -name "*.py" -exec grep -lP '\t' {} \;

# Count tabs in a file
tr -cd '\t' < file.txt | wc -c

# Count spaces vs tabs
echo "Spaces: $(tr -cd ' ' < file.txt | wc -c), Tabs: $(tr -cd '\t' < file.txt | wc -c)"
```

### Convert Tabs to Spaces (In-Place)

```bash
# Single file
sed -i 's/\t/    /g' file.txt

# All files in directory
find . -name "*.py" -exec sed -i 's/\t/    /g' {} \;

# Using expand (preserves alignment)
expand -t 4 file.txt > file.tmp && mv file.tmp file.txt
```

### Remove Trailing Whitespace

```bash
# sed
sed -i 's/[[:space:]]*$//' file.txt

# Find files with trailing whitespace
grep -rlP '\s+$' /path/to/project/
```

### Show Mixed Indentation

```bash
# Find files with both tabs and spaces for indentation
grep -rlP '^\t' . | xargs grep -lP '^ ' | sort -u

# Show lines with tab indentation
grep -nP '^\t' file.txt

# Show lines with space indentation
grep -nP '^ ' file.txt
```

## Comparison

| Tool | Tabs | Trailing spaces | EOL | Non-printable | In-place fix |
|------|------|-----------------|-----|---------------|--------------|
| `cat -A` | `^I` | visible via `$` | `$` | Yes | No |
| `sed -n 'l'` | `\t` | visible | `$` | Yes | No |
| `grep -P '\t'` | highlight | with `\s+$` | No | No | No |
| `vim :set list` | custom | custom | custom | Yes | Yes |
| `expand` | converts | No | No | No | Yes |
| `hexdump -C` | `09` | `20` | `0a`/`0d` | Yes | No |
