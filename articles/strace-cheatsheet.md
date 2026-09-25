# strace Cheatsheet

`strace` traces the **system calls** a process makes and the **signals** it receives. Because almost everything a program does — opening files, reading, writing, networking — goes through the kernel via syscalls, `strace` is the fastest way to answer "what is this process actually doing?": which files it opens (and which fail), where it hangs, what it talks to on the network, and where it spends time.

It works by attaching with `ptrace`, so it has overhead and needs permission to trace the target (root, or the same user with `yama` ptrace_scope allowing it). For block-I/O tracing see the [blktrace Guide](articles/blktrace-guide.md); for stuck processes, [Why Processes in D State Can't Be Killed](articles/linux-processes-d-state.md).

## Common System Calls to Watch For

Recognizing these in the output tells you what a program is doing:

| Syscall | What it does |
|---------|--------------|
| `open` / `openat` | Open a file for reading/writing |
| `access` | Check file existence/permissions |
| `read` / `write` | Read or write data on a descriptor |
| `close` | Close a file descriptor |
| `lseek` | Move the read/write offset within a file |
| `fstat` / `stat` | Retrieve file metadata (size, mode, times) |
| `fchmod` / `fchown` | Change permissions / ownership |
| `statfs` | Retrieve filesystem details |
| `connect` / `sendto` / `recvfrom` | Network connection and datagram I/O |
| `futex` | Fast userspace mutex (locking) — often noise |

## Key Options

| Option | Meaning |
|--------|---------|
| `-e trace=SET` | Trace only these syscalls (e.g. `-e trace=open,close`) |
| `-e SYSCALL` | Shorthand filter (e.g. `-e open`) |
| `-p PID` | Attach to a running process by PID |
| `-f` | Follow forked child processes (and threads) |
| `-F` | Also follow `vfork` (implied by `-f` on modern strace) |
| `-D` | Detach the tracer into the background after attaching |
| `-c` | Summary only: count calls, time, and errors per syscall |
| `-C` | Like `-c` but also print the regular trace |
| `-S KEY` | Sort the `-c` summary (e.g. `-S time`, `-S calls`) |
| `-w` | In the summary, measure **wall-clock** time, not CPU-in-syscall |
| `-o FILE` | Write the trace to FILE instead of stderr |
| `-t` / `-tt` / `-ttt` | Timestamp: HH:MM:SS / +µs / epoch seconds.µs |
| `-r` | Relative timestamp (delta since the previous syscall) |
| `-T` | Show time spent in each syscall |
| `-i` | Print the instruction pointer at each syscall |
| `-s SIZE` | Print up to SIZE bytes of string arguments (default 32) |
| `-P PATH` | Only trace syscalls that touch PATH |
| `-y` / `-yy` | Annotate descriptors with paths (`-yy` adds socket details) |
| `-x` | Print non-ASCII strings as hex |
| `-v` | Verbose — unabbreviated structs/argument lists |

## Basic Usage

```sh
strace ls                          # trace a command from start to finish
strace -o output.txt ls            # save the trace to a file
strace -f ./program                # follow child processes/threads
strace -s 80 ls                    # show up to 80 bytes of string args
```

## Filtering Which Syscalls

Tracing everything is noisy; filter to what matters:

```sh
strace -e open ls                          # only open() (shorthand)
strace -e trace=open,read ls /home         # only open and read
strace -e trace=file ls                    # all file-related syscalls (a class)
strace -e trace=network nc example.com 80  # all network syscalls
```

`strace` supports syscall **classes**: `file`, `process`, `network`, `signal`, `memory`, `desc` (descriptors). `-e trace=file` is often more useful than naming individual calls.

Modern strace prefixes class names with `%` (`%file`, `%network`, `%process`, `%memory`, `%desc`); the bare form (`file`) is the older, deprecated spelling. Both still work:

```sh
strace -e trace=%file ls                   # file-related syscalls (current form)
strace -e trace=%network curl -s host      # network syscalls
strace -e trace=%memory ./program          # mmap/brk/mprotect, etc.
```

### Excluding syscalls

Prefix the set with `!` to trace everything **except** those calls (escape it for the shell):

```sh
strace -e trace=\!futex ./program          # everything but futex
strace -e trace=\!write,read ./program     # exclude write and read
```

This is cleaner than `2>&1 | grep -v` when you know which calls to drop, since it also removes them from the `-c` summary.

### Filter out the noise

`futex` and similar calls can drown the output. Invert with `grep`:

```sh
strace -Tf ./program 2>&1 | grep -v futex
```

> `strace` writes to **stderr**, so redirect with `2>&1` before piping to `grep` (or use `-o file`). Forgetting the `2>&1` is the most common reason a `strace ... | grep` shows nothing.

### Filtering signals

Signal delivery is traced separately from syscalls with `-e signal=`:

```sh
strace -e signal=none ./program            # syscalls only, suppress signal lines
strace -e signal=SIGTERM,SIGKILL ./program # only these signals
strace -e trace=signal ./program           # signal-related syscalls (kill, rt_sigaction, ...)
```

## Attaching to a Running Process

```sh
strace -p 1234                     # attach to PID 1234 (Ctrl-C to detach)
strace -f -p 1234                  # include its threads/children
strace -s 128 -f -p 1234           # longer strings, with children
```

Detach cleanly with **Ctrl-C** — the traced process keeps running.

## Timing: Where Does the Time Go?

The `-c` summary is the go-to for "why is this slow?". Let it run, then Ctrl-C:

```sh
strace -c -p 11084                 # summarize a running process until Ctrl-C
strace -c ls >/dev/null            # summarize a command start to finish
strace -c -S time ls >/dev/null    # sort the summary by time spent
```

> **`-c` vs `-c -w`.** By default `-c` measures **CPU time spent inside** each syscall. Add `-w` to measure **wall-clock** time instead — crucial for I/O- or lock-bound calls that spend most of their time *blocked* (waiting), which barely register under plain `-c` but dominate under `-w`.

```sh
strace -c -w -p 11084              # rank syscalls by time actually waited
```

For per-call timing inline (rather than a summary):

```sh
strace -tt -T ls                   # µs timestamp + duration per call
strace -r ls                       # relative delta between calls (spot stalls)
strace -ttt ls                     # epoch seconds.µs timestamps
```

`-r` prints the time elapsed since the previous syscall — a quick way to eyeball where a program pauses.

## Practical Recipes

### What config files does a program read?

```sh
strace -e open php 2>&1 | grep php.ini
```

### Why can't the program find/open a file?

Look for an `open`/`openat` or `access` that returns an error (e.g. `ENOENT`, `EACCES`):

```sh
strace -e trace=open,access ./program 2>&1 | grep your-filename
```

### Trace a network connection

```sh
strace -e trace=connect,poll,select,sendto,recvfrom nc example.com 80
strace -e trace=network curl -s http://example.com >/dev/null
```

### Full, detailed capture to a file

A verbose capture with timestamps, durations, resolved descriptors, and long strings — useful for handing off a bug report:

```sh
strace -f -tt -T -v -yy -x -s 4096 -o /tmp/trace.txt ./program
# equivalent compact flag bundle:
strace -ttTvfo /tmp/su.strace su - appmon
```

- `-f` follow children · `-tt` µs timestamps · `-T` durations · `-v` verbose · `-yy` descriptor/socket details · `-x` hex for binary · `-s 4096` long strings · `-o` to a file.

### Trace a service restart (e.g. crond, clvmd, multipath)

```sh
strace -f -tt -T -v -x -s 4096 -o /tmp/multipath.txt multipath -v2 -ll
strace -ttfFv -o /tmp/crond.strace service crond restart
```

When tracing a **service restart**, the traced wrapper may linger in the background. Suspend it with **Ctrl-Z**, then kill the strace process so it doesn't hold the terminal:

```sh
# after Ctrl-Z:
killall -9 strace
```

## Reading the Output

Each line is `syscall(args) = return`. A negative return with an errno name explains failures.

```text
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=128708, ...}) = 0
openat(AT_FDCWD, "/etc/missing.conf", O_RDONLY) = -1 ENOENT (No such file or directory)
close(3)                                = 0
```

With `-tt` each line is prefixed by a timestamp:

```text
22:15:03.849122 openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
22:15:03.849151 read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0", 832) = 832
```

The `-c` summary is a table sorted by time, with an errors column:

```text
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 34.78    0.000023          23         1           execve
 21.74    0.000014           2         7           mmap
 13.04    0.000009           2         4           openat
  8.70    0.000006           1         4         1 stat
```

The **errors** column is where failed syscalls show up — a fast way to spot `ENOENT`/`EACCES` problems at a glance.

## Notes and Gotchas

- **Output goes to stderr.** Pipe with `2>&1 | ...` or capture with `-o file`.
- **Overhead is real.** `strace` slows the target significantly (every syscall is intercepted); don't leave it attached to a busy production process longer than needed.
- **Permissions.** Tracing another user's process needs root; on hardened systems `kernel.yama.ptrace_scope` may block attaching even as the same user.
- **Default string truncation is 32 bytes.** Bump `-s` (e.g. `-s 4096`) when you need to see full paths or payloads.
- **`strace` = syscalls; `ltrace` = library calls.** If you care about `libc`/library function calls rather than kernel syscalls, reach for `ltrace`.
- **Modern alternative.** On recent systems, `strace -k` adds stack traces; `perf trace` and `bpftrace` offer lower-overhead syscall tracing at scale.
- **Linux only.** `strace` is Linux-specific. On macOS use `sudo dtruss command` (or `sudo dtruss -f command` to follow children); on other BSDs, `ktrace`/`truss`. Note SIP restricts `dtruss` against system binaries.

## Quick Reference

```sh
strace ls                              # trace a command
strace -f -p 1234                      # attach to a running PID + threads
strace -e trace=open,read ls           # filter to specific syscalls
strace -e trace=file ./prog 2>&1 | grep -v futex   # file calls, drop noise
strace -c -p 1234                      # summarize time per syscall (Ctrl-C)
strace -tt -T ls                       # timestamp + duration per call
strace -f -tt -T -yy -s 4096 -o t.txt ./prog       # full capture to a file
strace -e trace=network nc example.com 80          # trace networking
```

## Links

- Red Hat: [How do I use strace to trace system calls made by a command?](https://access.redhat.com/articles/2483)

For related material, see the [blktrace Guide](articles/blktrace-guide.md), the [ps Cheatsheet](articles/ps-cheatsheet.md), [Why Processes in D State Can't Be Killed](articles/linux-processes-d-state.md), and the [fuser Cheatsheet](articles/fuser-cheatsheet.md).
