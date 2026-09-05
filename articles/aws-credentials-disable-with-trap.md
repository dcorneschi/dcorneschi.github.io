# Temporarily Disabling AWS Credentials Safely (with `$$` and `trap`)

A common need when working with AWS auth: you want to **prove that a role-based or SSO auth
path works without static keys**. The cleanest way is to make the AWS CLI/SDK stop seeing
`~/.aws/credentials` for a while, run your test, and then put the file back — guaranteed,
even if the test crashes or you Ctrl-C out.

Two small shell lines do exactly that:

```bash
mv ~/.aws/credentials ~/.aws/credentials.disabled.$$
trap "mv ~/.aws/credentials.disabled.$$ ~/.aws/credentials" EXIT
```

This article explains what each piece does, why it's written this way, and how to use it
without shooting yourself in the foot.

---

## 1. Why disable the credentials file at all?

The AWS CLI and SDKs resolve credentials through a **provider chain**, roughly in order:

1. Environment variables (`AWS_ACCESS_KEY_ID`, etc.)
2. The shared credentials file (`~/.aws/credentials`)
3. The shared config file (`~/.aws/config`) — SSO sessions, `credential_process`, assumed
   roles
4. Container/instance identity (ECS task role, EC2 instance profile, EKS IRSA / Pod
   Identity)

If `~/.aws/credentials` has static keys, it usually **wins over** the role/SSO paths lower
in the chain. So the fastest way to test "does my SSO profile / assumed role / instance
role actually work?" is to temporarily remove the static file from the equation and see if
the lower-priority provider takes over.

Renaming the file (rather than deleting it) means you keep the keys and can restore them.

---

## 2. Line 1 — disable, with a unique backup name

```bash
mv ~/.aws/credentials ~/.aws/credentials.disabled.$$
```

- **`mv`** renames the file. After this, `~/.aws/credentials` no longer exists, so the CLI
  falls through to the next provider in the chain.
- **`$$`** is a special shell variable that expands to the **PID (process ID) of the
  current shell**. If your shell's PID is `48213`, the backup becomes
  `~/.aws/credentials.disabled.48213`.

### Why `$$`?

It makes the backup filename **unique per shell** so you don't clobber an earlier backup.
Run the pattern in two different shells and each gets its own file
(`...disabled.48213`, `...disabled.49001`), instead of the second run overwriting the
first's saved credentials.

It's not guaranteed unique forever (PIDs get recycled over time), but within a working
session it's effectively unique and needs zero extra tooling. A timestamp is a readable
alternative:

```bash
mv ~/.aws/credentials ~/.aws/credentials.disabled.$(date +%Y%m%d-%H%M%S)
```

---

## 3. Line 2 — guarantee the restore with `trap`

```bash
trap "mv ~/.aws/credentials.disabled.$$ ~/.aws/credentials" EXIT
```

`trap` is a bash builtin that registers a command to run automatically when the shell
receives a **signal** or hits a specific **event**. General form:

```bash
trap 'COMMAND' SIGNAL_OR_EVENT
```

Here it says: **when the shell exits, move the backup back to `~/.aws/credentials`.**

### `EXIT` fires no matter how you leave

`EXIT` is a pseudo-event (not a real OS signal) that runs whenever the shell terminates —
normal finish, `exit`, an error, or Ctrl-C. That's the whole point: your credentials get
restored **even if the commands in between fail or you interrupt them**. Without the trap,
a crash would leave your static keys disabled and you'd have to remember to move them back
by hand.

### Why `$$` again, and why double quotes

- `$$` resolves to the **same PID** as line 1 (it's the same shell), so the trap references
  the exact backup file you created.
- With **double quotes**, `$$` is expanded **now**, when the trap is registered — the
  resolved filename (`...disabled.48213`) is baked into the handler. With single quotes it
  would expand when the trap fires instead; for `$$` in the same shell that's the same
  value, so both work here. Double quotes just make the concrete filename explicit.

---

## 4. Putting it together

The intended flow — a setup / auto-teardown idiom:

```bash
# 1. Disable static creds, keeping a unique backup
mv ~/.aws/credentials ~/.aws/credentials.disabled.$$

# 2. Ensure they're restored on exit, no matter what
trap "mv ~/.aws/credentials.disabled.$$ ~/.aws/credentials" EXIT

# 3. Run whatever must NOT see the static keys
aws sts get-caller-identity          # should now resolve via SSO / role / instance profile
# ... more commands, tests, a script ...

# 4. On exit, the trap runs automatically and puts ~/.aws/credentials back
```

`aws sts get-caller-identity` is the go-to verification: it prints the ARN of whoever you're
authenticated as, so you can confirm the fallback provider (SSO user, assumed role, IRSA
role, etc.) is really the one in effect.

---

## 5. Gotchas and safer variants

- **Interactive shell timing.** If you paste these two lines straight into your terminal,
  the `EXIT` trap won't fire until you **close that shell** — which might be much later than
  you meant. This idiom is cleanest inside a **script** or a **subshell**, where "exit"
  means "the task is done":

  ```bash
  (
    mv ~/.aws/credentials ~/.aws/credentials.disabled.$$
    trap "mv ~/.aws/credentials.disabled.$$ ~/.aws/credentials" EXIT
    aws sts get-caller-identity
  )   # subshell exits here → creds restored immediately
  ```

- **`SIGKILL` can't be trapped.** If the shell is killed with `kill -9`, the trap never
  runs and your file stays disabled. Same if the machine loses power. The backup still
  exists — you just restore it manually (see below).

- **The file might not exist.** If `~/.aws/credentials` isn't there, line 1's `mv` fails and
  you never made a backup, but the trap still tries to move a nonexistent
  `...disabled.$$` back. Guard it if you're scripting:

  ```bash
  [ -f ~/.aws/credentials ] && mv ~/.aws/credentials ~/.aws/credentials.disabled.$$
  trap '[ -f ~/.aws/credentials.disabled.'$$' ] && mv ~/.aws/credentials.disabled.'$$' ~/.aws/credentials' EXIT
  ```

- **Env vars still win.** Disabling the file does nothing if `AWS_ACCESS_KEY_ID` /
  `AWS_SECRET_ACCESS_KEY` are exported in your environment — they sit **above** the file in
  the chain. Unset them too if you want a clean test:

  ```bash
  unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
  ```

- **Don't stack multiple `EXIT` traps expecting them to combine.** A second
  `trap ... EXIT` **replaces** the first in bash. If you need several cleanup steps, put
  them in one handler (a function is tidiest).

- **Manual restore** if a trap never ran:

  ```bash
  ls ~/.aws/credentials.disabled.*                       # find your backup
  mv ~/.aws/credentials.disabled.48213 ~/.aws/credentials
  ```

---

## 6. Handy `trap` reference

| Event / signal | Fires when |
|---|---|
| `EXIT` | The shell terminates (any reason except SIGKILL) |
| `INT` | Ctrl-C (SIGINT) |
| `TERM` | `kill` (SIGTERM) |
| `ERR` | A command returns non-zero (most useful with `set -e`) |
| `HUP` | Terminal/connection closed (SIGHUP) |

| Command | Effect |
|---|---|
| `trap 'cmd' EXIT` | Register `cmd` to run on exit |
| `trap 'cleanup' EXIT INT TERM` | Same handler for several events |
| `trap -p` | Print currently registered traps |
| `trap - EXIT` | Remove the `EXIT` trap |

---

## Summary

- **Line 1** renames `~/.aws/credentials` out of the way so the AWS CLI/SDK falls through to
  a role/SSO provider; **`$$`** (the shell's PID) makes the backup filename unique so you
  don't overwrite a prior backup.
- **Line 2** uses **`trap ... EXIT`** to guarantee the file is restored whenever the shell
  exits — normal finish, error, or Ctrl-C — with `$$` referencing the exact backup created
  in line 1.
- Together they're a **setup / auto-teardown** idiom for testing role-based auth without
  static keys. Run it in a **script or subshell** for correct timing, remember `SIGKILL`
  bypasses traps, and unset `AWS_*` env vars if you want a truly clean test.

---

### Sources

- [Bash Reference Manual — `trap` and signals](https://www.gnu.org/software/bash/manual/bash.html#index-trap)
- [Bash special parameters (`$$` and friends)](https://www.gnu.org/software/bash/manual/bash.html#Special-Parameters)
- [Configuration and credential file settings (AWS CLI User Guide)](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)
- [Configure the AWS CLI credential provider chain / precedence](https://docs.aws.amazon.com/sdkref/latest/guide/standardized-credentials.html)
