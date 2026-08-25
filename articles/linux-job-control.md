# Linux Job Control

Manage background and foreground processes within a shell session. Job control lets you suspend, resume, and switch between multiple tasks without opening additional terminals.

## Starting Background Jobs

| Command | What It Does |
|---------|--------------|
| `./someprogram &` | Start a program in the background |
| `command &` | Run any command in the background |
| `nohup command &` | Run in background and survive shell exit (output goes to `nohup.out`) |
| `command > /dev/null 2>&1 &` | Run in background with all output discarded |

```bash
# Start a long-running process in the background
./build.sh &

# Start and immediately get the PID
./server &
echo "Server PID: $!"
```

## Suspending and Resuming

| Action | What It Does |
|--------|--------------|
| `Ctrl-Z` | Suspend (pause) the current foreground process and return to the shell |
| `bg` | Resume the most recently suspended job in the background |
| `bg %2` | Resume job number 2 in the background |
| `fg` | Bring the most recent background/suspended job to the foreground |
| `fg %1` | Bring job number 1 to the foreground |

### Typical Workflow

```bash
# Start a program
./long-task

# Realize you need the terminal — press Ctrl-Z
# [1]+  Stopped     ./long-task

# Continue it in the background
bg

# Later, bring it back to the foreground
fg
```

## Listing Jobs

| Command | What It Does |
|---------|--------------|
| `jobs` | Show all jobs in the current shell |
| `jobs -l` | Show jobs with process IDs |
| `jobs -p` | Show only PIDs of jobs |
| `jobs -r` | Show only running jobs |
| `jobs -s` | Show only stopped (suspended) jobs |

### Job Status Indicators

```
[1]+  Running                 ./server &
[2]-  Stopped                 vim config.yml
[3]   Done                    make build
```

| Symbol | Meaning |
|--------|---------|
| `+` | The current (default) job — what `fg` and `bg` act on without arguments |
| `-` | The previous job — becomes current if the current job finishes |
| (none) | Other jobs |

### Job States

| State | Meaning |
|-------|---------|
| Running | Actively executing in the background |
| Stopped | Suspended (paused) by Ctrl-Z or a signal |
| Done | Finished execution |
| Terminated | Killed by a signal |

## Job Specifiers

Use `%` prefixes to target specific jobs:

| Specifier | What It Targets |
|-----------|-----------------|
| `%1` | Job number 1 |
| `%+` or `%%` | The current job (same as bare `fg`/`bg`) |
| `%-` | The previous job |
| `%string` | Job whose command starts with `string` |
| `%?string` | Job whose command contains `string` |

```bash
# Bring the job that starts with "vim" to the foreground
fg %vim

# Kill the job containing "server" in its command
kill %?server
```

## Killing Jobs

| Command | What It Does |
|---------|--------------|
| `kill %1` | Send SIGTERM to job 1 |
| `kill -9 %1` | Force kill (SIGKILL) job 1 |
| `kill -STOP %1` | Suspend job 1 (same as Ctrl-Z) |
| `kill -CONT %1` | Resume a stopped job (same as `bg`) |

## Disowning Jobs

Remove a job from the shell's job table so it survives shell exit:

| Command | What It Does |
|---------|--------------|
| `disown %1` | Remove job 1 from job table (keeps running after shell exit) |
| `disown -h %1` | Mark job 1 to not receive SIGHUP on shell exit (stays in job table) |
| `disown -a` | Remove all jobs from job table |

```bash
# Start a long process, then decide you want to close the terminal
./backup.sh &
disown %1
# Safe to close the terminal now
```

## nohup vs disown vs &

| Method | Survives Shell Exit | Survives SIGHUP | Output Handling |
|--------|--------------------:|:---------------:|:----------------|
| `command &` | No | No | Connected to terminal |
| `command & disown` | Yes | Yes | Connected to terminal (may error on write) |
| `nohup command &` | Yes | Yes | Redirected to `nohup.out` |

- Use `&` alone for quick background tasks within the session
- Use `nohup command &` when you know upfront the job must survive
- Use `disown` when you've already started a job and realize you need to leave

## Process Monitoring

Pair job control with these commands to monitor system performance:

| Command | What It Does |
|---------|--------------|
| `ps xauww` | Show the full system process list |
| `free` | Show free and used memory |
| `vmstat 3` | Print system stats (CPU, memory, I/O) every 3 seconds |
| `iotop` | Show disk I/O by process (requires root, install with `yum install iotop`) |
| `top` | Interactive process viewer |
| `htop` | Enhanced interactive process viewer |

## Recipes

### Run something and get your terminal back immediately

```bash
make build > build.log 2>&1 &
echo "Build started in background, check build.log for output"
```

### Suspend vim to run a quick command, then return

```bash
# In vim, press Ctrl-Z
# Run your command
git status
# Return to vim
fg
```

### Move a running foreground process to the background

```bash
# Process is running... press Ctrl-Z to suspend
# [1]+  Stopped     ./long-running-task

# Resume it in the background
bg %1
# [1]+ ./long-running-task &
```

### Keep a process alive after SSH disconnect

```bash
# Option 1: nohup from the start
nohup ./deploy.sh > deploy.log 2>&1 &

# Option 2: already running — suspend, background, disown
# Ctrl-Z
bg
disown %1

# Option 3: use tmux or screen (better for interactive processes)
tmux new -s deploy
./deploy.sh
# Ctrl-B, D to detach
```

### Wait for background jobs to finish

```bash
# Start multiple jobs
./task1.sh &
./task2.sh &
./task3.sh &

# Wait for all background jobs
wait
echo "All tasks complete"

# Wait for a specific job
./slow-task.sh &
PID=$!
wait $PID
echo "slow-task finished with exit code $?"
```

## Gotchas

- **Jobs are per-shell** — background jobs belong to the shell session that started them. You can't see them from another terminal (use `ps` for that).
- **Ctrl-Z doesn't kill** — it suspends. The process is paused and consuming no CPU but still holds memory and open files.
- **Background jobs and terminal output** — a background job writing to the terminal can produce messy interleaved output. Redirect stdout/stderr to a file.
- **SIGHUP on shell exit** — by default, background jobs receive SIGHUP when you close the shell. Use `nohup` or `disown` to prevent this.
- **Stopped jobs prevent logout** — bash warns "There are stopped jobs" if you try to exit. Run `fg` and quit them, or `kill %job` them.
- **`$!` gives last background PID** — useful for scripting but only captures the most recent background process.

## Tips

- `jobs -l` is your best friend for seeing what's running and their PIDs.
- Use `Ctrl-Z` + `bg` as a quick "I forgot to add `&`" fix.
- For anything long-running over SSH, prefer `tmux` or `screen` over `nohup` — they let you reattach and see output.
- In scripts, use `wait` to synchronize parallel background tasks.
- Combine `&` with output redirection to keep your terminal clean: `cmd > out.log 2>&1 &`

## See Also

- [Bash Essentials Guide](articles/bash-essentials-guide.md) — shell fundamentals and scripting
- [ps Cheatsheet](articles/ps-cheatsheet.md) — viewing and filtering processes
- [top Cheatsheet](articles/top-cheatsheet.md) — interactive process monitoring
