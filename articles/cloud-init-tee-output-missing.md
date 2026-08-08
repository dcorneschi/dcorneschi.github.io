# cloud-init: Why tee Output Doesn't Appear in Logs

When using `tee` inside cloud-init scripts (`runcmd`, user-data shell scripts, or `bootcmd`), the output often doesn't appear where you expect it — or it disappears entirely. This is a common gotcha that trips up engineers who are used to `tee` working predictably in interactive shells.

## The Problem

You write something like this:

```yaml
#cloud-config
runcmd:
  - echo "hello" | tee /tmp/hello.txt
  - apt-get install -y nginx 2>&1 | tee -a /var/log/my-setup.log
```

And then find that:

- `/var/log/cloud-init-output.log` is missing the `tee`'d output
- The output appears in your custom log file but not on the console
- Or worse, the output appears nowhere and the command seems to silently fail

## Why This Happens

### cloud-init Already Pipes Through tee

The key thing most people miss: cloud-init's **default output configuration** already redirects all stdout and stderr through `tee`. The default config (typically in `/etc/cloud/cloud.cfg.d/05_logging.cfg`) is:

```yaml
output: { all: "| tee -a /var/log/cloud-init-output.log" }
```

This means cloud-init is already piping the entire execution through `tee -a /var/log/cloud-init-output.log`. When you add your own `tee` inside a `runcmd` entry, you're creating a **nested pipe within an already-redirected stream**.

### What Actually Happens

The execution chain looks like this:

```
Your command (echo "hello")
    → pipe to your tee (/tmp/hello.txt)
        → your tee's stdout goes to cloud-init's captured stdout
            → cloud-init pipes that through its own tee (-a /var/log/cloud-init-output.log)
```

In this case, `tee` works as expected — output lands in both your file and cloud-init's output log. But problems arise when:

1. **Buffering hides output** — when stdout is not a terminal (which it never is in cloud-init), programs use full buffering instead of line buffering. Output may not flush until the command finishes or the buffer fills up.

2. **You redirect stdout before tee** — if you do `command > /some/file 2>&1 | tee ...`, the `>` redirect takes priority over the pipe. The pipe receives nothing.

3. **List-form commands don't invoke a shell** — if you use the list format `[echo, "hello", "|", "tee", "/tmp/test"]`, cloud-init executes this via `execve()` directly. The `|` is passed as a literal argument to `echo`, not interpreted as a pipe.

4. **The output directive conflicts with your tee** — if you've customized the `output` directive to use `>` (overwrite) instead of `| tee -a`, your inner `tee` writes to its file, but cloud-init's redirect overwrites the output log on each stage.

## Common Scenarios and Fixes

### Scenario 1: tee in List-Form runcmd (Pipe Not Interpreted)

```yaml
# BROKEN — pipe is passed as literal argument, not interpreted by shell
runcmd:
  - [echo, "setup complete", "|", "tee", "/tmp/status.txt"]
```

The fix: use string form so a shell interprets the pipe:

```yaml
# WORKS — string form invokes /bin/sh -c
runcmd:
  - echo "setup complete" | tee /tmp/status.txt
```

### Scenario 2: Output Missing from cloud-init-output.log

```yaml
runcmd:
  - apt-get update 2>&1 | tee /var/log/apt-setup.log
```

This actually works correctly in most cases — output goes to both `/var/log/apt-setup.log` (via your `tee`) and `/var/log/cloud-init-output.log` (via cloud-init's outer `tee`). If you're not seeing it in `cloud-init-output.log`, check:

```bash
# Is the output directive configured correctly?
grep -r "output" /etc/cloud/cloud.cfg.d/

# Is the file being overwritten between stages?
ls -la /var/log/cloud-init-output.log
```

If the `output` directive uses `>` instead of `| tee -a` or `>>`, each stage overwrites the file. Only the last stage's output survives.

### Scenario 3: Buffering Delays Output

When piping through `tee` in a non-interactive environment, commands with large output (like `apt-get`) buffer internally. Output may not appear in real-time:

```yaml
runcmd:
  # Force line-buffered output with stdbuf
  - stdbuf -oL apt-get install -y nginx 2>&1 | tee -a /var/log/setup.log

  # Or use unbuffer (from expect package)
  - unbuffer apt-get install -y nginx 2>&1 | tee -a /var/log/setup.log

  # Or use script to fake a TTY
  - script -qc "apt-get install -y nginx" /dev/null | tee -a /var/log/setup.log
```

### Scenario 4: You Want Output in Both a Custom File and cloud-init-output.log

The simplest approach — don't use `tee` at all. Just let cloud-init capture the output naturally, and separately redirect to your file:

```yaml
runcmd:
  # Option A: tee to your file, cloud-init captures stdout automatically
  - apt-get install -y nginx 2>&1 | tee -a /var/log/setup.log

  # Option B: write to file separately, let cloud-init handle the log
  - bash -c 'apt-get install -y nginx 2>&1 | tee -a /var/log/setup.log'
```

Both produce the same result because cloud-init is already wrapping the entire execution.

### Scenario 5: Using tee with sudo Inside runcmd

`runcmd` already runs as root, so `tee` doesn't need `sudo`. But if you're writing to a path that requires special handling:

```yaml
# UNNECESSARY — runcmd runs as root
runcmd:
  - echo "setting" | sudo tee /etc/some/config

# SIMPLER — just use tee directly
runcmd:
  - echo "setting" | tee /etc/some/config
```

## The output Directive Explained

cloud-init's `output` key controls where stdout/stderr from each stage goes:

```yaml
#cloud-config
# Default: pipe all output through tee (append)
output: { all: "| tee -a /var/log/cloud-init-output.log" }

# Per-stage control
output:
  init:
    output: "> /var/log/cloud-init.out"
    error: "> /var/log/cloud-init.err"
  config: "tee -a /var/log/cloud-config.log"
  final:
    - ">> /var/log/cloud-final.out"
    - ">> /var/log/cloud-final.err"
```

The stages map to:

| Stage | What runs there |
|-------|----------------|
| `init` | Network stage — bootcmd, write_files, mounts, users, ssh |
| `config` | Config stage — locale, ntp, runcmd (writes script) |
| `final` | Final stage — packages, scripts_user (executes runcmd script) |

Your `runcmd` commands execute in the **final** stage. If you only redirect the `config` stage, `runcmd` output won't be captured.

## When tee Is Actually Useful in cloud-init

There are legitimate cases where adding `tee` inside cloud-init scripts makes sense:

```yaml
runcmd:
  # 1. Write to a custom log file AND let cloud-init capture it
  - bash -c 'my-installer.sh 2>&1 | tee -a /var/log/my-app-install.log'

  # 2. Write to a file that needs to exist for another process
  - echo "READY" | tee /run/myapp/init-complete

  # 3. Append to a config file (tee -a as a root-capable append)
  - echo "10.0.0.5 db.internal" | tee -a /etc/hosts

  # 4. Write to multiple files simultaneously
  - echo "node_id=abc123" | tee /etc/myapp/id /var/run/myapp/id
```

## When to Skip tee Entirely

For simple logging, you don't need `tee` — cloud-init is already capturing everything:

```yaml
runcmd:
  # These all appear in /var/log/cloud-init-output.log automatically
  - echo "Starting setup..."
  - apt-get update
  - apt-get install -y nginx
  - systemctl enable --now nginx
  - echo "Setup complete"
```

If you want a separate log file without needing real-time streaming, use plain redirection:

```yaml
runcmd:
  - apt-get install -y nginx >> /var/log/setup.log 2>&1
```

But note: this output will **not** appear in `cloud-init-output.log` because you've redirected stdout/stderr away from cloud-init's capture. The trade-off is yours to make.

## Debugging Missing Output

```bash
# 1. Check what output directive is active
grep -r "output" /etc/cloud/cloud.cfg.d/

# 2. Check if cloud-init-output.log exists and has content
ls -la /var/log/cloud-init-output.log
wc -l /var/log/cloud-init-output.log

# 3. Check if a stage used > (overwrite) instead of >> (append)
head -5 /var/log/cloud-init-output.log  # if only final-stage output, earlier stages were overwritten

# 4. Check the runcmd script that cloud-init generated
cat /var/lib/cloud/instance/scripts/runcmd

# 5. Re-run the script manually to see output
bash -x /var/lib/cloud/instance/scripts/runcmd
```

## Solutions

### Option 1: Redirect Both stdout and stderr to tee

The most common reason output is missing: `tee` writes to stdout, but your command's errors go to stderr. Always redirect both:

```bash
command 2>&1 | tee -a /var/log/your-log.log
```

### Option 2: Pipe to logger for syslog/journald Capture

```bash
command 2>&1 | tee -a /var/log/your-log.log | logger -t cloud-init
```

### Option 3: Use exec to Redirect All Output for an Entire Script

```bash
exec > >(tee -a /var/log/your-log.log)
exec 2>&1
```

This captures everything that follows — no need to pipe each command individually.

### Option 4: Append to cloud-init's Own Log

```bash
command 2>&1 | tee -a /var/log/cloud-init-output.log
```

## Best Practice: exec Redirect in runcmd

```yaml
#cloud-config
runcmd:
  - |
    exec > >(tee -a /var/log/my-init.log)
    exec 2>&1
    echo "Starting initialization..."
    # Your commands here
    echo "Initialization complete"
```

This ensures all subsequent commands in that script block have their output captured both in your custom log and visible to cloud-init's logging system.

## Complete Example

```yaml
#cloud-config
runcmd:
  - |
    # Set up logging for entire script
    exec > >(tee -a /var/log/custom-init.log)
    exec 2>&1
    
    echo "=== Starting Custom Initialization ==="
    date
    
    # Install packages
    apt-get update
    apt-get install -y docker.io
    
    # Configure services
    systemctl enable docker
    systemctl start docker
    
    echo "=== Initialization Complete ==="
    date
```

## Common Patterns

### Pattern 1: Simple Command Logging

```bash
echo "Installing Docker..." 2>&1 | tee -a /var/log/setup.log
apt-get install -y docker.io 2>&1 | tee -a /var/log/setup.log
```

### Pattern 2: Function with Logging

```bash
log_and_run() {
    echo "Running: $*" 2>&1 | tee -a /var/log/setup.log
    "$@" 2>&1 | tee -a /var/log/setup.log
}

log_and_run apt-get update
log_and_run apt-get install -y docker.io
```

### Pattern 3: Entire Script Block

```yaml
runcmd:
  - |
    {
      echo "Starting setup at $(date)"
      apt-get update
      apt-get install -y docker.io
      echo "Setup complete at $(date)"
    } 2>&1 | tee -a /var/log/setup.log
```

## Summary

The root cause is almost always one of these:

1. **List-form runcmd** — pipes aren't interpreted without a shell. Use string form.
2. **Buffering** — non-interactive `tee` buffers output. Use `stdbuf -oL` if you need real-time output.
3. **Redirect conflict** — `>` steals stdout before the pipe reaches `tee`. Check your redirect order.
4. **The output directive** — cloud-init already captures everything via `| tee -a`. You don't usually need to add your own.
5. **Stage mismatch** — if you customized `output` per-stage, make sure the `final` stage is configured (that's where `runcmd` scripts execute).

When in doubt: use string-form commands, skip `tee` for simple logging (cloud-init captures it), and add `tee` only when you need a custom file alongside cloud-init's log.
