# Running a Script Directly from a GitHub Gist

Gists are a quick way to share a shell script, and you can execute one straight from its raw URL without saving a file first. This guide covers the common invocation methods, the raw-URL format (including version pinning), cache-busting, and — most importantly — why "curl | bash" deserves caution and how to run it more safely.

## The Raw Gist URL

Scripts execute from the **raw** URL, not the gist web page:

```text
# Latest version of the file
https://gist.githubusercontent.com/{user}/{gist_id}/raw/{file}

# A specific revision, pinned by commit hash
https://gist.githubusercontent.com/{user}/{gist_id}/raw/{commit_hash}/{file}
```

Omitting the commit hash always serves the newest version — convenient, but it means the script can change under you. Pinning the hash guarantees you run exactly what you reviewed (see the security section).

## 1. curl Piped to bash

```bash
curl -L https://gist.githubusercontent.com/user/gist_id/raw/file | bash
```

`-L` follows redirects if the URL has moved. This is the classic one-liner, but the script's output and any prompts get muddled with the download — and you're executing code you haven't seen.

## 2. Process Substitution (bash/zsh)

```bash
bash <(curl -sL https://gist.githubusercontent.com/user/gist_id/raw/file)
```

`<(...)` feeds the downloaded script to bash as if it were a file. Unlike a pipe, this lets you **pass arguments** to the script:

```bash
bash <(curl -sL "$GIST_URL") arg1 arg2
```

`-s` silences curl's progress meter; `-L` follows redirects.

## 3. wget Instead of curl

```bash
bash <(wget -nv -O - https://gist.githubusercontent.com/user/gist_id/raw/file)
```

`-O -` writes to stdout instead of a file; `-nv` (no-verbose) quiets wget while keeping errors.

## 4. Cache-Busting

Raw gist URLs can be served from cache, so a fresh edit may not appear immediately. Append a throwaway query parameter to force a fresh fetch:

```bash
curl -s "https://gist.githubusercontent.com/user/gist_id/raw/file?_=$(uuidgen)" | bash
```

The `?_=<random>` value is ignored by the server but makes each request unique. `$(date +%s)` works as an alternative to `uuidgen`.

## Security: Review Before You Run

Piping a remote URL straight into a shell runs whatever that URL returns, with your privileges, sight unseen. Treat it like any untrusted code:

```bash
# 1. Download and READ the script first
curl -L https://gist.githubusercontent.com/user/gist_id/raw/file -o script.sh
less script.sh

# 2. Run it only after you've reviewed it
bash script.sh
```

Additional precautions:

- **Pin the commit hash** in the URL so the reviewed content can't change between your read and your run — a plain `curl | bash` against the latest version is vulnerable to the file being edited in between.
- **Never pipe to a privileged shell blindly.** Avoid `curl ... | sudo bash`; download, review, then run with elevation only if warranted.
- **Trust the source.** Only run gists from authors and accounts you trust; a gist ID is not proof of authorship.
- **Watch for partial-download execution.** With `curl | bash`, a dropped connection can execute a truncated script. Downloading to a file first avoids running half a script.

## Summary

- Run from the **raw** URL: `curl -L <raw-url> | bash`, or `bash <(curl -sL <raw-url>)` when you need to pass arguments.
- `wget -O -` is the wget equivalent of `curl` to stdout.
- Add `?_=$(uuidgen)` to dodge stale caches.
- Prefer download-review-run over blind piping, pin the commit hash, and never `| sudo bash` code you haven't read.
