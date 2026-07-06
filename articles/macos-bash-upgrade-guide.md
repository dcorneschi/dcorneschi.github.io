macOS ships with an older version of bash (3.2.x) due to licensing reasons. Here's how to install and use a newer version.

### Check Current Version

```bash
bash --version
```

The system bash is located at `/bin/bash` and is typically version 3.2.x.

### Install Newer Bash with Homebrew

#### 1. Install Homebrew (if needed)
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

#### 2. Install Latest Bash
```bash
brew install bash
```

#### 3. Check New Version
```bash
"$(brew --prefix)/bin/bash" --version
```

### Make It Your Default Shell

#### 1. Add New Bash to Allowed Shells (if not already added)
```bash
grep -qxF "$(brew --prefix)/bin/bash" /etc/shells || \
     sudo bash -c 'echo "$(brew --prefix)/bin/bash" >> /etc/shells'
```

#### 2. Change Default Shell
```bash
chsh -s "$(brew --prefix)/bin/bash" $USER
```

#### 3. Restart Terminal or Run
```bash
exec /opt/homebrew/bin/bash
```

### Verify the Change

```bash
# Check version
echo $BASH_VERSION

# Check which bash is being used
which bash

# Should show /opt/homebrew/bin/bash
```

### Notes

- The old system bash remains at `/bin/bash` (version 3.2.x)
- The new bash is installed at `$(brew --prefix)/bin/bash` (latest version)
- Using `$(brew --prefix)` makes the commands work on both Intel and Apple Silicon Macs
- You may need to restart your terminal or IDE for changes to take effect
- Some scripts may still reference `/bin/bash` explicitly in their shebang lines

### One-liner Installation

```bash
brew install bash && grep -qxF "$(brew --prefix)/bin/bash" /etc/shells || \
     sudo bash -c 'echo "$(brew --prefix)/bin/bash" >> \
     /etc/shells' && chsh -s "$(brew --prefix)/bin/bash" $USER
```
