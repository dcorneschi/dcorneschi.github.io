# Bash Pipelines and Redirections

Complete reference for pipelines and input/output redirection in Bash. Covers standard file descriptors, redirection operators, pipes, here-documents, process substitution, custom file descriptors, and advanced techniques.

## File Descriptors

When working with input and output in Bash, everything is treated as a stream of data. These streams are managed using file descriptors. By default, three file descriptors are always open in a shell session:

| File Descriptor | Abbreviation | Description |
|-----------------|--------------|-------------|
| 0 | STDIN | Standard input (usually your keyboard) |
| 1 | STDOUT | Standard output (usually your terminal screen) |
| 2 | STDERR | Standard error (error messages, also writes to screen) |

Unless explicitly closed, file descriptors always point to something — typically your terminal when Bash starts.

In addition to 0, 1, and 2, you can open and close custom file descriptors (3, 4, 5, ...) and even duplicate them to point to the same location.

## Redirection Operators

| Redirection | Description |
|-------------|-------------|
| `command > file` | Redirect stdout of a command to a file. Same as `command 1> file`. |
| `command 2> file` | Redirect stderr of a command to a file. |
| `command >> file` | Append stdout to a file without overwriting existing data. |
| `command 2>> file` | Append stderr to a file without overwriting existing data. |
| `command &> file` | Redirect both stdout and stderr to a file. Same as `command > file 2>&1`, but not `command 2>&1 > file`. |
| `command > /dev/null` | Discard stdout. `/dev/null` discards all data written to it. |
| `command 2> /dev/null` | Discard stderr of a command. |
| `command &> /dev/null` | Discard both stdout and stderr. Same as `command > /dev/null 2>&1`. |
| `command < file` | Redirect the contents of a file to stdin. Same as `command 0< file`. |
| `command << EOL` | Redirect multiple lines to stdin using a here-document. Ends when a line with only EOL is found. |
| `command <<- EOL` | Same as `<<` but strips leading tab characters from the input. |
| `command <<< 'string'` | Use a here-string to redirect input from a single string. |
| `command <<< $word` | Same as above but using a variable. |
| `command >\| file` | Override the noclobber shell option to force file overwrite. |
| `exec 2> file` | Redirect stderr of all commands to a file. |
| `exec > file` | Redirect stdout of all commands to a file. |
| `exec 3< file` | Create custom file descriptor (3) for reading from a file. |
| `exec 3> file` | Create custom file descriptor (3) for writing to a file. |
| `exec 3<> file` | Create custom file descriptor (3) for both reading and writing. |
| `exec 3>&-` | Close the created file descriptor. |
| `exec 4<&3` | Make file descriptor 4 a copy of file descriptor 3. |
| `exec 4>&3-` | Copy file descriptor 3 to 4 and close file descriptor 3. |
| `echo "foo" >&3` | Write to custom file descriptor 3. |
| `cat <&3` | Read from custom file descriptor 3. |
| `exec 3<> /dev/tcp/host/port` | Open a TCP connection. Bash feature only. |
| `exec 3<> /dev/udp/host/port` | Open a UDP connection. Bash feature only. |
| `(command1; command2) > file` | Redirect stdout from multiple commands to a file using a subshell. |
| `{ command1; command2; } > file` | Redirect stdout from multiple commands to a file without subshell. Faster. |
| `command < <(command1)` | Redirect stdout of command1 to an anonymous pipe and pass to command. |
| `command < <(command1) < <(command2)` | Redirect output of two commands to anonymous pipes and pass to another command. |
| `command1 \>(command2)` | Run command2 with stdin connected to a named pipe and pass filename to command1. |
| `command1 > >(command2)` | Run command2 with stdout from command1 through a named pipe. |
| `command1 \| command2` | Redirect stdout of command1 to stdin of command2. |
| `command1 \|& command2` | Redirect stdout and stderr of command1 to stdin of command2. (Bash 4.0+) |
| `command \| tee file` | Redirect stdout of command to both a file and screen. |
| `exec {filew}> file` | Open file for writing using a named file descriptor (filew). Bash 4.1+. |
| `command 3>&1 1>&2 2>&3` | Swap stdout and stderr of a command. |
| `cmd > >(cmd1) 2> >(cmd2)` | Send stdout to cmd1 and stderr to cmd2. |
| `cmd1 \| cmd2 \| cmd3 \| cmd4; echo ${PIPESTATUS[@]}` | Show exit codes of all piped commands. |
