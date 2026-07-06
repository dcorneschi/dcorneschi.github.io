Here's how to use `sed` to replace entire lines based on a string match:

### Basic Syntax
```bash
sed '/pattern/c\new line content' file
```

The `c\` command replaces the entire matched line with new content.

### Examples

#### Example 1: Replace a configuration line
```bash
# Replace any line containing "DEBUG" with a new debug setting
sed '/DEBUG/c\DEBUG=false' config.txt

# Before: DEBUG=true
# After:  DEBUG=false
```

#### Example 2: Replace a specific user in /etc/passwd format
```bash
# Replace entire line for user "olduser" with new user info
sed '/^olduser:/c\newuser:x:1001:1001:New User:/home/newuser:/bin/bash' userfile

# Before: olduser:x:1000:1000:Old User:/home/olduser:/bin/bash
# After:  newuser:x:1001:1001:New User:/home/newuser:/bin/bash
```

#### Example 3: Replace Docker service definition
```bash
# Replace any line containing "nginx:1.20" with updated version
sed '/nginx:1.20/c\    image: nginx:1.22-alpine' docker-compose.yml

# Before:     image: nginx:1.20
# After:      image: nginx:1.22-alpine
```

#### Example 4: Update SSH configuration
```bash
# Replace SSH port configuration line
sed -i.bak '/^#Port 22/c\Port 2222' /etc/ssh/sshd_config

# Before: #Port 22
# After:  Port 2222

# Disable password authentication
sed -i.bak '/PasswordAuthentication/c\PasswordAuthentication no' /etc/ssh/sshd_config

# Before: PasswordAuthentication yes
# After:  PasswordAuthentication no
```

#### Example 5: Update UFW firewall rules
```bash
# Replace UFW default policy line
sed -i.backup '/DEFAULT_INPUT_POLICY/c\DEFAULT_INPUT_POLICY="DROP"' /etc/default/ufw

# Before: DEFAULT_INPUT_POLICY="ACCEPT"
# After:  DEFAULT_INPUT_POLICY="DROP"

# Update UFW logging level
sed -i.backup '/LOGLEVEL/c\LOGLEVEL=medium' /etc/ufw/ufw.conf

# Before: LOGLEVEL=low
# After:  LOGLEVEL=medium
```

### Key Points

- Use `-i` flag to edit files in-place: `sed -i '/pattern/c\new content' file`
- **Create backup before editing**: `sed -i.bak '/pattern/c\new content' file`
- The pattern can be a regex: `sed '/^#.*TODO/c\# COMPLETED' file`
- Escape special characters in the replacement text if needed
- The `c\` command replaces the **entire** line, not just the matched part

### Backup Options

#### Create backup with extension
```bash
# Creates original.txt.bak before modifying original.txt
sed -i.bak '/DEBUG/c\DEBUG=false' original.txt
```

#### Create backup with timestamp
```bash
# Creates file with timestamp suffix
sed -i.$(date +%Y%m%d) '/version/c\version=2.0' app.conf
```

This is perfect for configuration updates where you need to replace complete lines rather than just portions of text.
