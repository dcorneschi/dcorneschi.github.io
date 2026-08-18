# Vim Search and Replace

Comprehensive reference for Vim's substitution command — ranges, flags, patterns, capture groups, expression replacements, and real-world refactoring examples.

## Basic Syntax

```vim
:[range]s/pattern/replacement/[flags]
```

## Range Specifications

### Basic Ranges

| Range | Description |
|-------|-------------|
| `:s/old/new/` | Current line only |
| `:%s/old/new/g` | Entire file |
| `:1,5s/old/new/g` | Lines 1 to 5 |
| `:.,+3s/old/new/g` | Current line plus next 3 lines |
| `:'<,'>s/old/new/g` | Visual selection |
| `:$s/old/new/g` | Last line only |
| `:.,$s/old/new/g` | Current line to end of file |

### Advanced Ranges

| Range | Description |
|-------|-------------|
| `:1,$s/old/new/g` | First to last line (same as `%`) |
| `:10;+5s/old/new/g` | Line 10, then 5 lines from there |
| `:/pattern/,/pattern/s/old/new/g` | Between two patterns |
| `:/^class/,/^}/s/old/new/g` | From "class" to closing brace |
| `:g/pattern/s/old/new/g` | All lines containing pattern |
| `:v/pattern/s/old/new/g` | All lines NOT containing pattern |

## Flags

### Standard Flags

| Flag | Description |
|------|-------------|
| `g` | Replace all occurrences on each line (global) |
| `c` | Confirm each replacement |
| `i` | Case insensitive |
| `I` | Case sensitive (override ignorecase) |
| `n` | Show number of matches without replacing |
| `p` | Print lines after substitution |
| `l` | Print lines with list characters |
| `#` | Print lines with line numbers |

### Flag Combinations

```vim
gc    " Global replacement with confirmation
gn    " Count all matches globally
gci   " Global, confirm, case-insensitive
```

## Pattern Matching

### Basic Metacharacters

| Pattern | Matches |
|---------|---------|
| `.` | Any single character except newline |
| `*` | Zero or more of preceding atom |
| `\+` | One or more of preceding atom |
| `\?` | Zero or one of preceding atom (optional) |
| `\{n}` | Exactly n of preceding atom |
| `\{n,}` | n or more of preceding atom |
| `\{,m}` | 0 to m of preceding atom |
| `\{n,m}` | n to m of preceding atom |

### Anchors and Boundaries

| Pattern | Matches |
|---------|---------|
| `^` | Start of line |
| `$` | End of line |
| `\<` | Start of word boundary |
| `\>` | End of word boundary |
| `\%^` | Start of file |
| `\%$` | End of file |
| `\%V` | Inside visual selection |

### Character Classes

| Pattern | Matches |
|---------|---------|
| `[abc]` | Any of a, b, or c |
| `[^abc]` | Any character except a, b, or c |
| `[a-z]` | Any lowercase letter |
| `[A-Z]` | Any uppercase letter |
| `[0-9]` | Any digit |
| `\d` | Any digit [0-9] |
| `\D` | Any non-digit |
| `\s` | Any whitespace character |
| `\S` | Any non-whitespace character |
| `\w` | Any word character [a-zA-Z0-9_] |
| `\W` | Any non-word character |
| `\a` | Any alphabetic character |
| `\l` | Any lowercase character |
| `\u` | Any uppercase character |

### Advanced Pattern Elements

| Pattern | Matches |
|---------|---------|
| `\|` | Alternation (OR operator) |
| `\(\)` | Grouping (capture groups) |
| `\%(\)` | Non-capturing group |
| `\_s` | Whitespace including newlines |
| `\_^` | Start of line (including after newline) |
| `\_$` | End of line (including before newline) |
| `\_.` | Any character including newline |
| `\zs` | Start of match (for replacement) |
| `\ze` | End of match (for replacement) |

## Replacement Techniques

### Basic Replacements

| Pattern | Inserts |
|---------|---------|
| `&` or `\0` | Entire matched text |
| `\1`, `\2`, etc. | Captured groups |
| `~` | Previous replacement string |
| `\r` | Carriage return |
| `\n` | Newline |
| `\t` | Tab character |
| `\\` | Literal backslash |

### Case Conversion in Replacement

| Pattern | Effect |
|---------|--------|
| `\U` | Convert to uppercase until `\E` or end |
| `\L` | Convert to lowercase until `\E` or end |
| `\u` | Convert next character to uppercase |
| `\l` | Convert next character to lowercase |
| `\E` | End case conversion |

### Expression Replacements

```vim
" Use \= for expression evaluation
:%s/\d\+/\=submatch(0) * 2/g              " Double all numbers
:%s/$/\=" (" . line(".") . ")"/g          " Add line numbers
:%s/\w\+/\=toupper(submatch(0))/g         " Convert to uppercase
:%s/pattern/\=MyFunction(submatch(0))/g   " Use custom function
```

## Magic Modes

### Very Magic Mode (\v)

```vim
" Enables more intuitive regex syntax (fewer backslashes)
:%s/\v(word1|word2)/replacement/g
:%s/\v\d{4}-\d{2}-\d{2}/DATE/g
:%s/\v<(\w+)>/<strong>\1</strong>/g
```

### Very No Magic Mode (\V)

```vim
" Treats most characters literally
:%s/\V$HOME/replacement/g
:%s/\V[special.chars]/replacement/g
```

## Advanced Examples

### Text Manipulation

```vim
" Remove trailing whitespace
:%s/\s\+$//g

" Remove leading whitespace
:%s/^\s\+//g

" Normalize whitespace (multiple spaces to single)
:%s/\s\+/ /g

" Remove blank lines
:g/^$/d

" Convert DOS line endings to Unix
:%s/\r$//g

" Add semicolon to end of lines (if not present)
:%s/\([^;]\)$/\1;/g

" Remove C-style comments
:%s/\/\/.*$//g

" Convert snake_case to camelCase
:%s/_\([a-z]\)/\u\1/g

" Convert camelCase to snake_case
:%s/\([a-z]\)\([A-Z]\)/\1_\l\2/g
```

### Code Transformations

```vim
" Add quotes around unquoted strings
:%s/\v(:\s*)([^"'\s][^,}\]]*)/\1"\2"/g

" Remove console.log statements
:%s/^\s*console\.log.*$\n//g

" Convert single quotes to double quotes
:%s/'/"/g

" Convert function declarations to arrow functions (JS)
:%s/function \(\w\+\)(/const \1 = (/g

" Add 'const' to variable declarations
:%s/let \(\w\+\)/const \1/g

" Convert Python 2 print to Python 3
:%s/print \(.*\)$/print(\1)/g
```

### Advanced Pattern Matching

```vim
" Match email addresses
:%s/\v[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/[EMAIL]/g

" Match URLs
:%s/https\?:\/\/[^\s]\+/[URL]/g

" Match hex colors
:%s/#\x\{6\}/[COLOR]/g

" Swap function parameters
:%s/function(\([^,]\+\),\s*\([^)]\+\))/function(\2, \1)/g

" Extract filename from paths
:%s/.*\/\([^\/]\+\)$/\1/g
```

## Zero-Width Assertions

```vim
" Positive lookahead (match X followed by Y, don't include Y)
:%s/\(pattern\)\ze\(lookahead\)/replacement/g

" Positive lookbehind (match Y preceded by X, don't include X)
:%s/\(lookbehind\)\zs\(pattern\)/replacement/g
```

## Conditional Replacements

```vim
" Replace only on lines containing another pattern
:g/contains_this/s/old/new/g

" Replace only on lines NOT containing pattern
:v/contains_this/s/old/new/g

" Complex conditional with multiple patterns
:g/class.*public/s/private/protected/g
```

## Working with Registers

```vim
" Use register contents in replacement
:%s/pattern/\=@a/g                    " Use register 'a'
:%s/pattern/\=getreg('+')/g           " Use clipboard register
```

## Advanced Expression Replacements

```vim
" Increment numbers
:%s/\d\+/\=submatch(0) + 1/g

" Mathematical operations
:%s/\(\d\+\) \* \(\d\+\)/\=submatch(1) * submatch(2)/g

" Reverse words
:%s/\w\+/\=reverse(submatch(0))/g

" Replace with word length
:%s/\w\+/\=len(submatch(0))/g
```

## Working with Multiple Files

```vim
" Replace in all buffers
:bufdo %s/old/new/g | update

" Replace in all windows
:windo %s/old/new/g

" Replace in all tabs
:tabdo %s/old/new/g

" Replace in files matching pattern
:args *.txt | argdo %s/old/new/g | update
```

## Data Processing

```vim
" CSV: swap columns
:%s/\([^,]*\),\([^,]*\),\(.*\)/\2,\1,\3/g

" Add quotes around CSV fields
:%s/\([^,]*\)/"\1"/g

" Convert JSON keys
:%s/"\([^"]*\)":/\1:/g

" Extract URLs from HTML
:%s/.*href="\([^"]*\)".*/\1/g
```

## Log File Processing

```vim
" Extract timestamps from logs
:%s/^\[\([^\]]*\)\].*/\1/g

" Remove log levels
:%s/^\[.*\]\s*\(INFO\|DEBUG\|ERROR\):\s*//g

" Convert log format
:%s/\(\d\{4}-\d\{2}-\d\{2}\) \(\d\{2}:\d\{2}:\d\{2}\)/[\1T\2]/g
```

## HTML/XML Processing

```vim
" Convert HTML class to className (React)
:%s/<\(\w\+\)\s\+class="\([^"]*\)"/<\1 className="\2"/g

" Remove HTML tags but keep content
:%s/<[^>]*>//g

" Convert self-closing tags
:%s/<\(\w\+\)\([^>]*\)>/<\1\2\/>/g
```

## Configuration File Updates

```vim
" Update version numbers (increment patch)
:%s/version\s*=\s*"\([0-9]\+\)\.\([0-9]\+\)\.\([0-9]\+\)"/version = "\1.\2.\=submatch(3) + 1"/g

" Convert old config format to new
:%s/\v^(\w+)\s*:\s*(.*)$/\1 = \2/g

" Add default values to empty config
:%s/\v^(\w+)\s*=\s*$/\1 = "default"/g
```

## Database Queries

```vim
" Convert SELECT * to specific fields
:%s/SELECT \*/SELECT id, name, email/g

" Add table aliases
:%s/FROM \(\w\+\)/FROM \1 AS \l\1/g

" Convert JOINs to explicit syntax
:%s/JOIN \(\w\+\)/INNER JOIN \1/g
```

## Useful Functions for Expressions

### String Functions

| Function | Description |
|----------|-------------|
| `submatch(0)` | Entire match |
| `submatch(1)` | First capture group |
| `strlen(str)` | String length |
| `strpart(str, start, len)` | Substring |
| `substitute(str, pat, rep, flags)` | Nested substitution |
| `toupper(str)` | Convert to uppercase |
| `tolower(str)` | Convert to lowercase |
| `reverse(str)` | Reverse string |

### Math Functions

| Function | Description |
|----------|-------------|
| `abs(num)` | Absolute value |
| `pow(x, y)` | x to the power of y |
| `sqrt(num)` | Square root |
| `floor(num)` | Round down |
| `ceil(num)` | Round up |

### System Functions

| Function | Description |
|----------|-------------|
| `line('.')` | Current line number |
| `col('.')` | Current column number |
| `expand('%')` | Current filename |
| `strftime(format)` | Current timestamp |

## Scripting and Automation

### In Vim Scripts

```vim
function! CleanupCode()
    %s/\s\+$//ge
    %s/\r//ge
    %s/\t/    /ge
endfunction

autocmd BufWritePre *.py call CleanupCode()
```

### Command Line Integration

```bash
# Use vim for batch processing
vim -c ':%s/old/new/g | wq' file.txt

# Process multiple files
find . -name "*.txt" -exec vim -c ':%s/old/new/g | wq' {} \;
```

## Performance Tips

### For Large Files

```vim
" Disable syntax highlighting
:syntax off

" Use very magic mode for complex patterns
:%s/\v(complex|pattern)/replacement/g

" Use global command for sparse matches
:g/rare_pattern/s/old/new/g

" Combine multiple substitutions
:%s/old1/new1/g | %s/old2/new2/g | %s/old3/new3/g
```

### Using External Tools

```vim
" Use sed for complex operations
:%!sed 's/pattern/replacement/g'

" Use perl for advanced regex
:%!perl -pe 's/pattern/replacement/g'
```

## Quick Reference

| Command | Description |
|---------|-------------|
| `:s//~/` | Replace with previous replacement |
| `:&` | Repeat last substitution |
| `g&` | Repeat on all lines |
| `:s/pattern/\=@a/` | Replace with register a |
| `:s/\(.*\)/\1\1/` | Duplicate line content |
| `:%s/$/,/` | Add comma to end of each line |
| `:%s/^/> /` | Quote lines (email style) |
| `:%s/pattern//gn` | Count matches without replacing |

## Troubleshooting

### Common Errors

| Error | Cause |
|-------|-------|
| `E486: Pattern not found` | Check pattern syntax and case |
| `E554: Syntax error in pattern` | Check regex syntax |
| `E35: No previous substitute` | No previous substitution to repeat |

### Debugging Patterns

```vim
" Test pattern without replacing (count only)
:%s/pattern//gn

" Highlight matches
:set hlsearch
/pattern

" Confirm each replacement interactively
:%s/pattern/replacement/gc
```

### Recovery

```vim
u          " Undo last substitution
:e!        " Undo all changes to buffer
```

## Tips

1. Always test patterns with `/pattern` before substituting
2. Use `:set hlsearch` to visualize matches
3. Start with small ranges, then expand to `%`
4. Use `gc` flag for confirmation on important changes
5. `&` repeats the last substitution on current line
6. Use `\zs` and `\ze` for precise match boundaries
7. Combine with `:g` and `:v` for conditional replacements
8. Use `\=` expression replacements for dynamic content
9. Consider `\v` (very magic) for complex patterns to reduce backslashes
10. Always backup before complex substitutions on important files
