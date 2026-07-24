# Linux Processes and Signals Cheatsheet

## Process I/O and Return Codes

A process takes standard input (STDIN) and returns:

- **STDOUT** (standard output) — gets printed on your console
- **STDERR** (standard error) — you see that too unless you redirect it with `process 2> /dev/null` (more on that later)
- **Return code** — `0` on success, a different number otherwise

Of course there are processes that:

- Do not read STDIN (but instead read files, get data from the kernel, ...)
- Do not write to STDOUT (because they don't have anything to say)
- Do not write to STDERR (because there are no errors)

But every process returns a return code.

---

## Signals

Each signal has a default action, usually one of the following:

| Action | Description |
|--------|-------------|
| **Term** | Cause a program to terminate (exit) at once |
| **Core** | Cause a program to save a memory image (core dump), then terminate |
| **Stop** | Cause a program to stop executing (suspend) and wait to continue (resume) |

Programs can be prepared for expected event signals by implementing handler routines to ignore, replace, or extend a signal's default action.

### Sending Signals

Users signal their current foreground process by typing a keyboard control sequence:

| Shortcut | Action |
|----------|--------|
| `Ctrl-z` | Suspend (stop) the process |
| `Ctrl-c` | Kill (terminate) the process |
| `Ctrl-\` | Core dump the process |

To signal a background process or processes in a different session requires a signal-sending command.

---

## Commands

### Show processes in a specific state

- **R** (Running) — actively executing on CPU or waiting in the run queue
- **D** (Uninterruptible sleep) — waiting on I/O (usually disk), cannot be interrupted by signals
- **S** (Sleeping) — waiting for an event to complete (interruptible)
- **T** (Stopped) — suspended by a signal (e.g. `Ctrl-z`) or being traced by a debugger
- **Z** (Zombie) — process has finished but its parent hasn't read its exit code yet. Once the parent calls `wait`, the zombie disappears.

```bash
ps -eo state,pid,cmd | grep "^R"   # running
ps -eo state,pid,cmd | grep "^D"   # uninterruptible sleep, usually IO
ps -eo state,pid,cmd | grep "^S"   # sleeping
ps -eo state,pid,cmd | grep "^T"   # stopped
ps -eo state,pid,cmd | grep "^Z"   # zombie
```

### Count processes by state

```bash
ps -e h -o stat | sort | uniq -c | sort -rn
```

### Count processes by user

```bash
ps -ef | awk '{print $1}' | sort | uniq -c | sort -rn
```

### Display processor assignment (PSR column)

```bash
ps -eF
```

### Load average — find R and D state processes

```bash
# Linux
ps -eLo state,pid,cmd | grep -E '^[DR]'
# or
ps -eLo state,pid,cmd | grep -E '^(D|R)'
# or
top -H -b -n1 | awk '$8=="R" || $8=="D"'
```

### Top 10 processes by memory (RSS)

```bash
ps -e -o pid,comm,pmem,rss --sort -rss | head
```

### Top 10 processes by CPU

```bash
ps -eo %cpu,pid,user,cmd --sort=-%cpu | head
# or
top -b -n1 | sed -n '7,17'p
```

### Sort apache processes by RSS

```bash
ps -ylC httpd --sort:rss
```

### List open file descriptors by process

```bash
ls -d /proc/[1-9]*/fd/* 2>/dev/null | sed 's/\/fd.*$//' | uniq -c | sort -rn | head
```

### Display process hierarchy

```bash
# For a specific process
pstree -s 70544

# Full hierarchy
ps -Hwfe
# or
ps -ef --forest
# or
ps -afx
```

### Find the process ID of a running program

```bash
pidof httpd
```

### Sort processes by start time

```bash
ps -ef --sort=start_time
# or
ps aux --sort=lstart
```

### Monitoring "D" state processes

```bash
# Watch for "D" processes for one minute
for i in $(seq 1 60); do ps -eo state,pid,cmd | grep "^D"; echo "--- $i ---"; sleep 1; done

# or use watch
watch -n 1 "(ps aux | awk '\$8 ~ /D/ { print \$0 }')"
```

### Killing processes

```bash
# Find processes matching a pattern (preview before killing)
pgrep -f [part_of_a_command]       # add "-l" for long listing

# Kill processes matching a pattern
pkill -f [part_of_a_command]

# Kill with a loop
for pid in $(ps -ef | awk '/some search/ {print $2}'); do kill -9 $pid; done
```

### Show threads for a specific process

```bash
ps -Lp <pid>
```

### Show threads with thread IDs for all processes

```bash
ps -eLf
```

### Processes sorted by number of threads

```bash
ps -eo nlwp,pid,cmd --sort=-nlwp | head
```

### Processes that have been running the longest

```bash
ps -eo etime,pid,cmd --sort=-etime | head
```

### Show process tree with thread count

```bash
ps -efT --forest | head -40
```

### Show all processes with their nice value

```bash
ps -eo pid,ni,cmd
```

### Find processes owned by a specific user

```bash
ps -u username -o pid,%cpu,%mem,cmd
```

### Show only process name and PID

```bash
ps -eo pid,comm
```

### Show processes with their cgroup

```bash
ps -eo pid,cgroup,cmd
```

### Show security context (SELinux)

```bash
ps -eZ
```

### Processes using more than X% CPU

```bash
ps -eo %cpu,pid,cmd | awk '$1 > 5.0'
```

### Show elapsed time and CPU time side by side

```bash
ps -eo pid,etime,cputime,cmd --sort=-cputime | head
```

### Find multi-threaded processes

```bash
ps -eo nlwp,pid,cmd | awk '$1 > 1' | sort -rn | head
```

### Wide output (don't truncate command)

```bash
ps -efww
```

### Show process start time in full format

```bash
ps -eo pid,lstart,cmd | head
```

### Find processes without a controlling terminal (daemons)

```bash
ps -eo tty,pid,cmd | grep "^?"
```
