# awk Cheatsheet

awk is a pattern-scanning and text-processing language. It processes input line by line, splitting each line into fields, and applies pattern-action rules.

## Syntax

```bash
awk 'pattern { action }' file
awk -f script.awk file
awk 'BEGIN { action } pattern { action } END { action }' file
```

### Syntax Elements

| Element | Description |
|---------|-------------|
| `awk 'cmds' file(s)` | Invokes the awk commands on the file(s) |
| `$1 $2 $3...` | First, second, third fields |
| `$0` | Entire current line |
| `{.....}` | Executable step (print, assignments, getline) |
| `{print...}` | Prints to screen unless output is redirected |
| `(...)` | Test for patterns (if, while, etc.) |
| `awk -f prog inputf` | Read awk program from file |
| `NF` | Number of fields in current line |
| `{printf(...)}` | Print using user-supplied format |
| `BEGIN{...}` | Execute before processing input |
| `END{...}` | Execute after processing all input |
| `length(field)` | Count characters in a word or field |
| `#` | Comment in an awk program file |
| `array[countr]` | Associative array |
| `/string/` | Match current line for string |
| `~/string/` | Match for string or substring |
| `!~ /string/` | Match for anything not containing string |

## Command Line Options

```bash
-F fs          # Set field separator (default: whitespace)
-v var=value   # Set variable before execution
-f script      # Read AWK script from file
-W posix       # Enable POSIX mode
```

## Built-In Variables

### Field Variables

| Variable | Description |
|----------|-------------|
| `$0` | Entire record/line |
| `$1, $2, $3...` | Fields 1, 2, 3, etc. |
| `$NF` | Last field |
| `$(NF-1)` | Second to last field |

### Record Variables

| Variable | Description |
|----------|-------------|
| `NR` | Number of records processed so far |
| `FNR` | Record number in current file |
| `NF` | Number of fields in current record |
| `FILENAME` | Current filename |
| `RS` | Record separator (default: newline) |
| `FS` | Field separator (default: whitespace) |
| `OFS` | Output field separator (default: space) |
| `ORS` | Output record separator (default: newline) |

### Other Variables

| Variable | Description |
|----------|-------------|
| `ARGC` | Number of command line arguments |
| `ARGV` | Array of command line arguments |
| `ENVIRON` | Environment variables array |
| `RSTART` | Start position of match for `match()` |
| `RLENGTH` | Length of match for `match()` |

## Printing Fields

```bash
awk '{print $1}'
awk '{print $1, $3}'
awk '{print $1 " --- " $3}'
awk -F: '{print $2}'
awk '{print $NF}'
awk 'NR==1 {print $NF}'
awk '/regular-expression-to-match/ {print $1}'
awk '!/regular-expression-to-match/ {print $1}'
awk 'BEGIN{FS=","}{print $1}' db.txt
awk 'BEGIN{FS=",";OFS=" $ "}{print $1,$4}' db.txt
echo "This is how it works" | awk 'BEGIN{RS=" "}{print $0}'
```

## Field Separators

```bash
# Colon as field separator
awk -F: '{print $1}' /etc/passwd

# Multiple separators
awk -F'[,:]' '{print $1}' file

# Using BEGIN block
awk 'BEGIN{FS=","}{print $1}' data.csv

# Set both input and output separator
awk 'BEGIN{FS=",";OFS=" $ "}{print $1,$4}' data.csv

# List UID and GID separated by "-"
awk -F: 'BEGIN{OFS="-"} {print $3,$4}' /etc/passwd

# Space as record separator
echo "This is how it works" | awk 'BEGIN{RS=" "}{print $0}'
```

## Patterns

### Basic Patterns

```bash
/pattern/           # Lines matching regex pattern
$1 == "value"       # Field 1 equals "value"
$1 != "value"       # Field 1 not equals "value"
$1 ~ /pattern/      # Field 1 matches pattern
$1 !~ /pattern/     # Field 1 doesn't match pattern
NR == 1             # First line only
NR > 5              # Lines after line 5
NF > 3              # Lines with more than 3 fields
```

### Range Patterns

```bash
/start/,/end/       # Lines from start pattern to end pattern
NR==2,NR==5         # Lines 2 through 5
```

### Special Patterns

```bash
BEGIN { }           # Execute before processing any records
END { }             # Execute after processing all records
```

## Operators

### Arithmetic

| Operator | Description |
|----------|-------------|
| `+` | Addition |
| `-` | Subtraction |
| `*` | Multiplication |
| `/` | Division |
| `%` | Modulus |
| `^` or `**` | Exponentiation |
| `++` | Increment |
| `--` | Decrement |

### Comparison

| Operator | Description |
|----------|-------------|
| `==` | Equal to |
| `!=` | Not equal to |
| `<` | Less than |
| `<=` | Less than or equal to |
| `>` | Greater than |
| `>=` | Greater than or equal to |
| `~` | Match regex |
| `!~` | Don't match regex |

### Logical

| Operator | Description |
|----------|-------------|
| `&&` | AND |
| `\|\|` | OR |
| `!` | NOT |

### Assignment

| Operator | Description |
|----------|-------------|
| `=` | Assignment |
| `+=` | Add and assign |
| `-=` | Subtract and assign |
| `*=` | Multiply and assign |
| `/=` | Divide and assign |
| `%=` | Modulus and assign |

## Built-In Functions

### String Functions

```bash
length(string)              # Length of string
substr(string, start, len)  # Substring
index(string, substring)    # Position of substring
split(string, array, sep)   # Split string into array
gsub(regex, replacement, string)  # Global substitute
sub(regex, replacement, string)   # Single substitute
match(string, regex)        # Match regex, sets RSTART and RLENGTH
sprintf(format, ...)        # Formatted string
tolower(string)            # Convert to lowercase
toupper(string)            # Convert to uppercase
```

### Mathematical Functions

```bash
int(x)          # Integer part
sqrt(x)         # Square root
exp(x)          # Exponential
log(x)          # Natural logarithm
sin(x)          # Sine
cos(x)          # Cosine
atan2(y,x)      # Arctangent of y/x
rand()          # Random number 0-1
srand([seed])   # Seed random number generator
```

### I/O Functions

```bash
print           # Print current record
print $0        # Print current record
print $1, $2    # Print fields 1 and 2
printf(format, ...)  # Formatted print
getline         # Read next line
getline var     # Read next line into variable
getline < file  # Read from file
close(file)     # Close file
system(command) # Execute shell command
```

## Control Structures

### Conditionals

```bash
if (condition) {
    action
} else if (condition) {
    action
} else {
    action
}

condition ? true_action : false_action  # Ternary operator
```

### Loops

```bash
# For loop
for (i = 1; i <= 10; i++) {
    action
}

# For-in loop (arrays)
for (key in array) {
    action
}

# While loop
while (condition) {
    action
}

# Do-while loop
do {
    action
} while (condition)
```

### Loop Control

| Statement | Description |
|-----------|-------------|
| `break` | Exit loop |
| `continue` | Skip to next iteration |
| `next` | Skip to next input record |
| `exit` | Exit AWK program |
| `exit expression` | Go to END action; if in END, exit with status |

## Control Flow Reference

| Command | Description |
|---------|-------------|
| `{statements}` | Execute all statements in brackets |
| `if (expression) statement` | If true, execute statement |
| `if (expression) stmt1 else stmt2` | If/else |
| `while (expression) statement` | While loop |
| `for (expr1; expr2; expr3) statement` | C-style for loop |
| `for (variable in array) statement` | Iterate over array |
| `do statement while (expression)` | Do-while loop |
| `break` | Leave innermost loop |
| `continue` | Start next iteration |
| `next` | Start next input record |
| `exit` | Go to END action |
| `exit expression` | Exit with status code |

## Arrays

```bash
# Associative arrays
array[key] = value
array["name"] = "John"
array[1] = "first"

# Check if key exists
if (key in array) { ... }

# Delete array element
delete array[key]

# Delete entire array
delete array
```

## Regular Expressions

```bash
/pattern/           # Basic regex
/^pattern/          # Start of line
/pattern$/          # End of line
/[abc]/            # Character class
/[a-z]/            # Range
/[^abc]/           # Negated class
/./                # Any character
/.*/               # Zero or more any character
/.+/               # One or more any character
/.?/               # Zero or one any character
/(pattern1|pattern2)/  # Alternation
```

## Metacharacters Reference

| Metacharacter | Meaning |
|---------------|---------|
| `\` | Escape sequence (e.g., `\t` = tab, `\*` = literal *) |
| `^` | Beginning of string |
| `$` | End of string |
| `.` | Any single character |
| `[ABDU]` | Character class — matches A, B, D, or U; supports ranges like `[a-z]` |
| `A\|B` | Alternation — matches A or B |
| `DF` | Concatenation — matches D immediately followed by F |
| `R*` | Zero or more Rs |
| `R+` | One or more Rs |
| `R?` | Zero or one R |
| `NR==10,NR==25` | Range — lines 10 through 25 |

## Escape Sequences

| Sequence | Meaning |
|----------|---------|
| `\b` | Backspace |
| `\f` | Form feed |
| `\n` | Newline (line feed) |
| `\r` | Carriage return |
| `\t` | Tab |
| `\ddd` | Octal value (1-3 digits between 0 and 7) |
| `\c` | Any other character literally (`\\`, `\"`, `\*`) |

## Practical Examples

### File and Directory Operations

```bash
# List files larger than 810K
ls -al | awk '$5 > 828640 {print $9}'

# List files owned by mysql larger than 9K
ls -al | awk '$3 == "mysql" && $5 > 9355 {print $9}'

# List files owned by oracle larger than 512 bytes
ls -al | awk '$3 == "oracle" && $5 > 512 {print $9}'

# List files whose size is exactly 512 bytes
ls -al | awk '$5 == 512 {print $9}'

# List files whose size is greater than zero
ls -al | awk '$5 > 0 {print $9}'

# List files not owned by root
ls -al | awk '$3 != "root" {print $9}'

# Find largest file (recursive)
ls -lR | awk '{print $5 "\t" $9}' | sort -n | tail -1

# Print user and group for mysql-owned files
ls -l /var/log | awk '/mysql/ {print $3 " " $4}'

# Calculate directory size in MB
ls -al | awk '{total += $5} END{print "Total size: " total/1024/1024 " Mb"}'

# Including subdirectories
ls -lR | awk '{total += $5} END{print "Total size: " total/1024/1024 " Mb"}'

# Sum JPG file sizes
ls -l *.jpg | awk '{s += $5} END{print "Total size: " s}'

# Sort output and eliminate duplicates
awk '{print $1, $5, $7}' file | sort -u
```

### Log Analysis

```bash
# Top 20 popular pages from Apache log
cat /var/log/httpd/access_log | awk '{print $7}' | sort | uniq -c | sort -rn | head -20

# Display lines with HTTP 404
awk '$9 == 404 {print $0}' /var/log/httpd/access_log

# Count error lines
awk '/Error/ {nlines = nlines + 1} END {print nlines}' /var/log/messages
```

### User Administration

```bash
# Users with GID > 500
awk -F: '$3 > 500' /etc/passwd

# Check for a string in field
awk '$1 ~/nagios/' /etc/passwd

# Print lines containing multiple users
awk '/dani|nico|pisu/' /etc/passwd

# Print lines between patterns
awk '/500/, /502/' /etc/passwd

# Users where GID > UID
awk -F: '$4 > $3 {print $1}' /etc/passwd

# Users with same UID and GID
awk -F: '$3 == $4' /etc/passwd

# Users with bash shell
cat /etc/passwd | awk -F: '{if ($7 ~ /bash/) print $1}'

# Users without bash shell
cat /etc/passwd | awk -F: '{if ($7 !~ /bash/) print $1}'

# Display line number and field count
awk -F: '{print NR, "->", NF}' /etc/passwd

# Erase second field
awk -F: '{$2 = ""; print}' /etc/passwd

# Print line number and first field
awk -F: '{print NR, $1}' /etc/passwd

# Sum UIDs
awk -F: '{s += $3} END {print "Total sum is", s}' /etc/passwd

# List UID and GID with custom separator
awk -F: 'BEGIN{OFS="-"} {print $3,$4}' /etc/passwd
```

### Process and System

```bash
# Print process state and PID
ps aux | awk '{print $8 " " $2}'

# Display available memory
free -m | awk '/^Mem:/ {print $7 " MB"}'

# Print echo 100 times
yes | head -100 | awk '{print "echo"}'

# Display "D" state processes
top -b -n 1 | awk '{if (NR <=7) print; else if ($8 == "D") {print; count++}} END{print "Total status D: " count}'

# Change max password age for users with UID >= 500
awk -F: '$3 >= 500 {system("chage -M 180 " $1)}' /etc/passwd
```

### String Operations

```bash
# Print from nth field onward
echo 'This is a test' | awk '{print substr($0, index($0,$3))}'

# Replace first column
awk '{$1 = "ORACLE"; print}' data_file

# Remove first column
awk '{$1 = ""; print}' data_file

# Swap first two columns
awk '{temp = $1; $1 = $2; $2 = temp; print}' file

# Convert to uppercase
awk '{print toupper($0)}' file

# Print length of each line
awk '{print length($0)}' file

# Replace all occurrences
awk '{gsub(/old/, "new"); print}' file
```

### Mathematical Operations

```bash
# Multiply fields
awk '{print $1 * $2}' file

# Average
awk '{avg += $1; count++} END {print avg/count}' file

# Maximum value
awk '{if($1 > max) max = $1} END {print max}' file

# Column statistics (mean and standard deviation)
awk '{sum+=$1; sumsq+=$1*$1} END {print "Mean:", sum/NR, "StdDev:", sqrt(sumsq/NR - (sum/NR)^2)}' file
```

## Control Flow Examples

### if/else

```bash
awk '{if ($1 > 100) print "High:", $0}' file

awk '{
    if ($3 > 90) grade = "A"
    else if ($3 > 80) grade = "B"
    else grade = "C"
    print $1, grade
}' scores.txt

awk '{if ($2 ~ /error/) print "ERROR:", $0; else print "OK:", $0}' log.txt
```

### while

```bash
awk '{
    i = 1
    while (i <= NF) {
        print $i
        i++
    }
}' file

awk '{
    sum = 0; i = 1
    while (i <= NF) { sum += $i; i++ }
    print "Sum:", sum
}' numbers.txt
```

### for (C-style)

```bash
awk '{
    for (i = 1; i <= NF; i++)
        print i, $i
}' file

# Reverse field order
awk '{
    for (i = NF; i >= 1; i--)
        printf "%s ", $i
    print ""
}' file

# Print squares of 1-10
awk 'BEGIN {
    for (i = 1; i <= 10; i++)
        print i, i*i
}'
```

### for...in (arrays)

```bash
# Word frequency count
awk '{
    for (i = 1; i <= NF; i++)
        count[$i]++
}
END {
    for (word in count)
        print word, count[word]
}' file
```

### do...while

```bash
awk 'BEGIN {
    i = 1
    do {
        sum += i
        i++
    } while (i <= 100)
    print "Sum 1-100:", sum
}'
```

### break

```bash
# Print fields until "STOP"
awk '{
    for (i = 1; i <= NF; i++) {
        if ($i == "STOP") break
        print $i
    }
}' file
```

### continue

```bash
# Sum only positive numbers
awk '{
    sum = 0
    for (i = 1; i <= NF; i++) {
        if ($i < 0) continue
        sum += $i
    }
    print sum
}' file
```

### next

```bash
# Skip comment lines
awk '/^#/ {next} {print}' file

# Print odd-numbered lines only
awk 'NR % 2 == 0 {next} {print}' file
```

### exit

```bash
# Stop at first ERROR
awk '{print} /ERROR/ {exit}' file

# Print first 100 lines
awk '{if (NR > 100) exit; print}' file

# Exit with status code
awk '{
    count++
    if (count > 10) exit 1
} END {
    print "Processed", count, "lines"
}' file
```

## Escape Sequence Examples

```bash
awk '/\t/ {print}' file              # Lines containing tabs
awk '{gsub(/\t/, " "); print}' file  # Replace tabs with spaces
awk 'BEGIN {print "Line1\nLine2"}'   # Print with newline
awk 'BEGIN {print "Col1\tCol2\tCol3"}' # Print with tabs
awk '/\\/ {print}' file              # Lines with backslash
awk 'BEGIN {print "\101\102\103"}'   # Print ABC using octal
```

## Comparison Operator Examples

```bash
# Numeric comparisons
awk '$3 > 100 {print}' file
awk '$2 <= 50 {print}' file
awk '$1 == 42 {print}' file
awk '$5 >= 10 && $5 < 20 {print}' file

# String comparisons with regex
awk '$1 ~ /^[A-Z]/ {print}' file
awk '$2 !~ /test/ {print}' file
awk '$0 ~ /error|warning/ {print}' file
awk '$3 ~ /\.txt$/ {print}' file

# String equality
awk '$1 == "admin" {print}' file
awk '$2 != "NULL" {print}' file

# Combined conditions
awk '$1 > 50 && $2 ~ /active/ {print}' file
awk '$3 < 10 || $4 == "error" {print}' file
```

## Metacharacter Examples

```bash
# \ (escape)
awk '/\$[0-9]+/ {print}' file       # Literal $ followed by digits
awk '/\*\*\*/ {print}' file          # Literal ***

# ^ (beginning)
awk '/^Error/ {print}' log.txt
awk '$1 ~ /^[A-Z]/ {print}' file

# $ (end)
awk '/\.txt$/ {print}' file
awk '$2 ~ /ing$/ {print}' file

# . (any character)
awk '/c.t/ {print}' file             # Matches cat, cot, c9t
awk '/^....$/ {print}' file          # Exactly 4 characters

# [class]
awk '/[aeiou]/ {print}' file         # Lines with vowels
awk '/[0-9]/ {print}' file           # Lines with digits

# | (alternation)
awk '/error|warning/ {print}' log.txt

# * (zero or more)
awk '/go*d/ {print}' file            # gd, god, good, goood

# + (one or more)
awk '/go+d/ {print}' file            # god, good (not gd)

# ? (zero or one)
awk '/colou?r/ {print}' file         # color or colour
awk '/https?/ {print}' file          # http or https

# Range
awk 'NR==10, NR==25 {print}' file
awk '/START/, /END/ {print}' file
```

## Advanced Examples

```bash
# Remove duplicates (preserves order)
awk '!seen[$0]++' file

# Print every nth line
awk 'NR % 3 == 1' file

# Print lines in reverse order
awk '{lines[NR] = $0} END {for(i=NR; i>0; i--) print lines[i]}' file

# Group and count
awk '{count[$1]++} END {for(key in count) print key, count[key]}' file

# Count lines, words, characters
awk 'END {print NR}' file
awk '{words += NF} END {print words}' file
awk '{chars += length($0) + 1} END {print chars}' file

# Lines where field 1 is numeric
awk '$1 ~ /^[0-9]+$/' file

# Print line numbers
awk '{print NR ": " $0}' file

# Complex processing with error handling
awk '{
    if (NF < 3) {
        print "ERROR: Not enough fields" > "/dev/stderr"
        next
    }
    sum = 0
    for (i = 1; i <= NF; i++) {
        if ($i !~ /^[0-9]+$/) continue
        sum += $i
    }
    print "Sum:", sum
}' file

# Array processing with threshold
awk '{
    for (i = 1; i <= NF; i++)
        words[$i]++
}
END {
    for (word in words) {
        if (words[word] > 5)
            print word, words[word]
    }
}' file
```

## Formatting Output

```bash
# Number each line with printf
awk '{printf("%5d : %s\n", NR, $0)}' file

# Number only non-blank lines
awk 'NF{$0=++a " :" $0};1' file

# Line number per file
awk '{print FNR "\t" $0}' files*

# Line numbers across all files
awk '{print NR "\t" $0}' files*

# Double space a file
awk '1;{print ""}' file
awk 'BEGIN{ORS="\n\n"};1' file

# Double space (skip existing blank lines)
awk 'NF{print $0 "\n"}' file

# Triple space
awk '1;{print "\n"}' file
```

## One-Liner Reference

| Command | Description |
|---------|-------------|
| `awk '/pattern/ {print $1}'` | Standard Unix shells |
| `awk '/pattern/ {print "$1"}'` | DJGPP, Cygwin |
| `awk "/pattern/ {print \"$1\"}"` | GnuWin32, UnxUtils, Mingw |
| `awk '{print $NF}'` | Print last field |
| `awk '{field = $NF} END{print field}'` | Last field of last line |
| `awk 'END{print NR}'` | Count lines (`wc -l`) |
| `awk 'NF > 4'` | Lines with more than 4 fields |
| `awk '$NF > 4'` | Lines where last field > 4 |
| `awk '{print NF ":" $0}'` | Field count and line |
| `awk '{s=0; for (i=1; i<=NF; i++) s=s+$i; print s}'` | Sum fields per line |
| `awk '{for (i=1; i<=NF; i++) s=s+$i} END{print s}'` | Sum all fields |
| `awk '{for (i=1; i<=NF; i++) if ($i < 0) $i = -$i; print}'` | Absolute value |
| `awk '{for (i=1; i<=NF; i++) $i = ($i < 0) ? -$i : $i; print}'` | Absolute value (ternary) |
| `awk '{total = total + NF} END{print total}' file` | Total fields in file |
| `awk '/Beth/{n++} END{print n+0}' file` | Count lines with "Beth" |
| `awk '$1 > max {max=$1; maxline=$0} END{print max, maxline}'` | Largest first field |
| `awk '!seen[$0]++'` | Remove duplicates |
| `awk '{$1=$1}1'` | Remove leading/trailing whitespace |
| `awk '1;{print ""}'` | Double space |
| `awk 'BEGIN{ORS="\n\n"};1'` | Double space |
| `awk 'NF{print $0 "\n"}'` | Double space (skip blanks) |
| `awk '1;{print "\n"}'` | Triple space |
| `awk '{print FNR "\t" $0}' files*` | Line number per file |
| `awk '{print NR "\t" $0}' files*` | Line number all files |
| `awk '{printf("%5d : %s\n", NR,$0)}'` | Formatted line numbers |
| `awk 'NF{$0=++a " :" $0};1'` | Number non-blank lines |

## Tips and Tricks

1. **Multiple actions**: Use semicolons or newlines to separate
2. **Default action**: If no action specified, `{print}` is assumed
3. **Variable initialization**: Variables auto-initialize to 0 or ""
4. **Regex in variables**: Store patterns in variables for reuse
5. **Output redirection**: `print > "file"` or `print | "command"`
6. **POSIX vs GNU AWK**: Some features differ between implementations

## Common Gotchas

- Field assignments can change NF
- Changing `$0` recalculates all fields
- String comparison vs numeric comparison
- Regular expressions are case-sensitive by default
- Division by zero returns inf or -inf
- Uninitialized variables are 0 in numeric context, "" in string context
