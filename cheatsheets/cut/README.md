# cut Cheatsheet

`cut` extracts sections from each line of input — by character position, byte position, or delimited fields. It's one of the simplest and fastest text-processing tools on Linux.

---

## Syntax

```bash
cut OPTION... [FILE...]
```

| Option | Description |
|--------|-------------|
| `-d DELIM` | Use DELIM as the field delimiter (default: TAB) |
| `-f LIST` | Select fields (requires `-d`) |
| `-c LIST` | Select characters by position |
| `-b LIST` | Select bytes by position |
| `--complement` | Invert the selection (print everything except specified) |
| `-s` | Suppress lines that don't contain the delimiter (with `-f`) |
| `--output-delimiter=STR` | Use STR as the output delimiter |
| `-z` | Use null byte (`\0`) as the line delimiter instead of newline (coreutils 8.25+, RHEL 8+, Ubuntu 18.04+) |

---

## Field-Based Cutting (-f)

Extract fields from delimited data (CSV, TSV, colon-separated, etc.):

```bash
# Extract first field (default delimiter: TAB)
cut -f1 file.txt

# Extract fields 1 and 3
cut -f1,3 file.txt

# Extract fields 1 through 4
cut -f1-4 file.txt

# Extract field 3 onwards
cut -f3- file.txt

# Extract up to field 2
cut -f-2 file.txt
```

### With Custom Delimiter (-d)

```bash
# Colon-separated (e.g., /etc/passwd)
cut -d: -f1 /etc/passwd              # usernames
cut -d: -f1,3 /etc/passwd            # username and UID
cut -d: -f1,6 /etc/passwd            # username and home dir
cut -d: -f7 /etc/passwd              # login shells

# Comma-separated (CSV)
cut -d, -f1 data.csv                 # first column
cut -d, -f2,4 data.csv               # columns 2 and 4

# Space-delimited
cut -d' ' -f1 file.txt

# Semicolon-delimited
cut -d';' -f2 file.txt

# Pipe-delimited
cut -d'|' -f3 file.txt

# Dot-delimited (e.g., IP addresses)
echo "192.168.1.100" | cut -d. -f1   # 192
echo "192.168.1.100" | cut -d. -f4   # 100
```

---

## Character-Based Cutting (-c)

Extract specific character positions:

```bash
# First 5 characters
cut -c1-5 file.txt

# Single character at position 3
cut -c3 file.txt

# Characters 1, 3, and 5
cut -c1,3,5 file.txt

# Characters from position 10 onwards
cut -c10- file.txt

# First 20 characters
cut -c-20 file.txt

# Characters 5 to 10
cut -c5-10 file.txt
```

---

## Byte-Based Cutting (-b)

Same as `-c` for single-byte encodings; differs for multi-byte (UTF-8):

```bash
# First 10 bytes
cut -b1-10 file.txt

# Bytes 5 through 15
cut -b5-15 file.txt
```

> **Note:** `-c` operates on characters (respects locale/multi-byte). `-b` operates on raw bytes. For ASCII data they're identical. For UTF-8 with multi-byte characters, use `-c`.

---

## Complement (--complement)

Print everything **except** the specified fields/characters:

```bash
# Print all fields EXCEPT field 2
cut -d: -f2 --complement /etc/passwd

# Print all characters EXCEPT positions 1-5
cut -c1-5 --complement file.txt

# Print everything but characters 10 through 25
cut --complement -c10-25 /etc/motd

# Print everything but bytes 10 through 45
cut --complement -b10-45 /etc/motd

# Remove field 3 from CSV
cut -d, -f3 --complement data.csv
```

---

## Suppress Lines Without Delimiter (-s)

By default, `cut -f` prints lines that don't contain the delimiter unchanged. Use `-s` to skip them:

```bash
# Only output lines that contain a colon
cut -d: -f1 -s file.txt

# Useful for filtering header/comment lines that don't match the delimiter
grep -v "^#" config | cut -d= -f2
# Or:
cut -d= -f2 -s config
```

---

## Output Delimiter (--output-delimiter)

Change the delimiter in the output:

```bash
# Replace colons with tabs in output
cut -d: -f1,3,6 --output-delimiter=$'\t' /etc/passwd

# Convert : to , (entire line)
cut --output-delimiter=, -d: -f1- /etc/passwd

# Replace commas with pipes
cut -d, -f1-3 --output-delimiter='|' data.csv

# Replace tabs with commas (TSV to CSV)
cut -f1-5 --output-delimiter=',' file.tsv

# Convert spaces to commas in ps output (squeeze first to avoid empty fields)
ps -elf | tr -s ' ' | cut --output-delimiter=, -d' ' -f1-
```

---

## Practical Examples

### /etc/passwd

```bash
# List all usernames
cut -d: -f1 /etc/passwd

# List username:UID:GID
cut -d: -f1,3,4 /etc/passwd

# List username and shell
cut -d: -f1,7 /etc/passwd

# Find users with /bin/bash
grep "/bin/bash" /etc/passwd | cut -d: -f1

# List home directories
cut -d: -f6 /etc/passwd
```

### CSV Processing

```bash
# Extract header (first line) field names
head -1 data.csv | cut -d, -f1-5

# Get column 2 from all rows (skip header)
tail -n +2 data.csv | cut -d, -f2

# Extract multiple columns and change delimiter
cut -d, -f1,3,5 --output-delimiter=$'\t' data.csv
```

### Log Files

```bash
# Extract IP addresses from Apache access log (first field, space-delimited)
cut -d' ' -f1 /var/log/apache2/access.log

# Extract timestamp (field 4, removing bracket)
cut -d' ' -f4 /var/log/apache2/access.log | cut -c2-

# Extract date from syslog
cut -c1-15 /var/log/syslog

# Extract process name from syslog
cut -d' ' -f5 /var/log/syslog | cut -d: -f1
```

### Networking

```bash
# Extract IPs from ss output (use awk since ss uses variable spacing)
ss -tn | awk '{print $5}' | cut -d: -f1

# Extract IP from ifconfig
ifconfig | grep "inet " | cut -d' ' -f2

# Get hostname from FQDN
echo "server.example.com" | cut -d. -f1
# Output: server

# Get domain from FQDN
echo "server.example.com" | cut -d. -f2-
# Output: example.com

# Extract port from host:port
echo "192.168.1.1:8080" | cut -d: -f2
# Output: 8080

# Extract email domain
echo "user@example.com" | cut -d@ -f2
# Output: example.com

# Extract email username
echo "user@example.com" | cut -d@ -f1
# Output: user
```

### PATH Variable

```bash
# List each directory in PATH on its own line
echo "$PATH" | tr ':' '\n'
# Or using cut with output delimiter:
echo "$PATH" | cut -d: -f1- --output-delimiter=$'\n'

# Get the first directory in PATH
echo "$PATH" | cut -d: -f1

# Get the last directory in PATH
echo "$PATH" | rev | cut -d: -f1 | rev
```

### File Extensions

```bash
# Get file extension
echo "document.tar.gz" | rev | cut -d. -f1 | rev    # gz
echo "photo.jpg" | rev | cut -d. -f1 | rev          # jpg

# Get filename without extension
echo "photo.jpg" | rev | cut -d. -f2- | rev         # photo
echo "archive.tar.gz" | rev | cut -d. -f2- | rev    # archive.tar
```

---

## Combining with Other Commands

```bash
# cut + sort + uniq — count unique values in a field
cut -d, -f3 data.csv | sort | uniq -c | sort -rn

# cut + grep — filter then extract
grep "ERROR" app.log | cut -d' ' -f1-3

# cut from history — extract command (skip line number)
history | cut -d' ' -f4-

# cut from ps — convert spaces to commas
# NOTE: ps uses variable spacing, so squeeze spaces first with tr -s
ps -elf | tr -s ' ' ',' > processes.csv

# Or use ps with custom format (cleanest for CSV)
ps -eo pid,user,%cpu,%mem,comm --no-headers | tr -s ' ' ',' > processes.csv

# Or use awk (handles variable spacing natively)
ps -elf | awk '{$1=$1}1' OFS=',' > processes.csv

# cut + head — extract field from first line only
head -1 /etc/passwd | cut -d: -f1

# cut + find with null delimiter (-z for null-terminated input)
find . -name "*.txt" -print0 | cut -z -d/ -f2-

# cut + awk — when cut isn't enough (multiple spaces)
# cut can't handle multiple consecutive delimiters as one
# Use awk instead:
ps aux | awk '{print $1, $2, $11}'

# cut from command output
df -h | cut -c1-40
mount | cut -d' ' -f1,3,5

# cut + xargs — use extracted field as argument
cut -d: -f1 /etc/passwd | xargs -I{} id {}
```

---

## Limitations

| Limitation | Workaround |
|-----------|------------|
| Cannot handle multiple consecutive delimiters as one | Normalize first: `tr -s ' ' < file \| cut -d' ' -f2` |
| Cannot reorder fields (field order in output matches input) | Use `awk '{print $3, $1}'` |
| Single-character delimiter only | Use `awk -F'pattern'` for multi-char delimiters |
| No regex delimiter support | Use `awk` or `sed` |
| Cannot cut from the end (no negative positions) | Use `rev \| cut \| rev` trick |
| Does not understand CSV quoting | Use a CSV-aware parser (csvtool, Miller, Python csv) |

### Normalizing Repeated Spaces

`cut -d' '` treats each single space as a delimiter. Repeated spaces create empty fields:

```bash
# Problem: repeated spaces → empty fields
echo "alice   25   engineer" | cut -d' ' -f2
# Output: (empty — because field 2 is between two adjacent spaces)

# Solution: squeeze spaces first with tr -s
echo "alice   25   engineer" | tr -s ' ' | cut -d' ' -f2
# Output: 25
```

### CSV Quoting Limitation

`cut` does not understand CSV quoting rules — it splits on every delimiter character, even inside quotes:

```bash
printf '"web,public",80,active\n' | cut -d, -f1
# Output: "web    (wrong — split on the comma inside quotes)

# Use a CSV-aware tool instead for real CSV with quoted fields
```

---

## Common Errors

```bash
# "you must specify a list of bytes, characters, or fields"
# Cause: forgot -f, -c, or -b
cut file.txt                      # ❌ missing selection mode
cut -f1 file.txt                  # ✅

# "fields are numbered from 1"
# Cause: used field 0
cut -d: -f0 /etc/passwd           # ❌ fields start at 1
cut -d: -f1 /etc/passwd           # ✅

# "the delimiter must be a single character"
# Cause: multi-character delimiter
cut -d'::' -f1 file               # ❌ only one character allowed
cut -d: -f1 file                  # ✅
# For multi-char delimiters, use: awk -F'::' '{print $1}'
```

---

## Multiple Files and stdin

```bash
# Process multiple files (output is concatenated)
cut -d: -f1 /etc/passwd /etc/group

# Read from stdin explicitly
cat /etc/passwd | cut -d: -f1
# Or:
cut -d: -f1 -              # - means stdin

# Mix files and stdin
cut -d: -f1 file1.txt - file2.txt
# Reads file1, then stdin, then file2
```

---

## cut vs awk vs sed

| Task | cut | awk | sed |
|------|-----|-----|-----|
| Extract fixed fields from delimited data | ✅ Best | Works | Overkill |
| Multiple spaces as delimiter | ❌ | ✅ Best | ❌ |
| Reorder fields | ❌ | ✅ | ❌ |
| Conditional extraction | ❌ | ✅ | ✅ |
| Character ranges | ✅ Best | Works | Works |
| Performance (large files) | ✅ Fastest | Moderate | Moderate |

---

## Quick Reference

```bash
# Fields
cut -d: -f1 file              # first field, colon delimiter
cut -d, -f1,3 file            # fields 1 and 3, comma delimiter
cut -d' ' -f2- file           # field 2 onwards, space delimiter
cut -f1-3 file                # fields 1-3, tab delimiter (default)

# Characters
cut -c1-10 file               # first 10 characters
cut -c5 file                  # 5th character only
cut -c10- file                # from 10th character to end

# Modifiers
cut -d: -f2 --complement file        # everything except field 2
cut -d: -f1,3 --output-delimiter=$'\t' file  # change output delimiter
cut -d: -f1 -s file                  # suppress lines without delimiter

# Common patterns
cut -d: -f1 /etc/passwd              # list usernames
cut -d, -f2 data.csv                 # second CSV column
echo "$PATH" | cut -d: -f1           # first PATH directory
echo "a.b.c" | cut -d. -f2-          # b.c (domain from FQDN)
echo "file.txt" | rev | cut -d. -f1 | rev  # get extension
```
