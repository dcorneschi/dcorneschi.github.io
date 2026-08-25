# ansible-lint Guide

ansible-lint checks playbooks, roles, and collections for coding standards, deprecated patterns, and common mistakes. It catches issues that `--syntax-check` misses — like using `shell` instead of a proper module, missing task names, or non-idempotent commands.

## Install

```bash
# pip (recommended — works on any system)
pip3 install ansible-lint

# RHEL/Fedora (EPEL or AAP subscription)
sudo dnf install ansible-lint

# Ubuntu/Debian
sudo apt install ansible-lint

# From source (requires pip >= 22.3.1)
pip3 install git+https://github.com/ansible/ansible-lint

# Verify
ansible-lint --version
```

## Basic Usage

```bash
# Lint a single playbook
ansible-lint playbook.yml

# Lint multiple files
ansible-lint playbook.yml site.yml

# Lint an entire directory (roles, playbooks, etc.)
ansible-lint .

# Lint with a specific profile
ansible-lint --profile=production playbook.yml

# List all available rules
ansible-lint -L

# List rules for a specific tag
ansible-lint -L -t command-shell

# Show only rule names (for skip_list)
ansible-lint -L | awk '{print $1}'

# Lint and auto-fix what's possible
ansible-lint --fix playbook.yml

# Offline mode (skip checks requiring network)
ansible-lint --offline playbook.yml
```

## Profiles

Profiles group rules by strictness. Each profile includes all rules from the profiles below it:

```
production  ← strictest (AAP certified/validated content)
    │
 shared     ← Galaxy publishing requirements
    │
 safety     ← security and non-determinism
    │
 moderate   ← readability and naming conventions
    │
 basic      ← common coding issues and formatting
    │
  min       ← fatal errors only (Ansible can't load the file)
```

| Profile | Purpose | Key Rules |
|---------|---------|-----------|
| `min` | Ansible can load the file | `syntax-check`, `parser-error`, `load-failure` |
| `basic` | Prevent common mistakes | `command-instead-of-module`, `no-free-form`, `name`, `yaml`, `jinja` |
| `moderate` | Readability standards | `name[casing]`, `name[template]`, `spell-var-name` |
| `safety` | Security concerns | `risky-file-permissions`, `package-latest`, `risky-shell-pipe` |
| `shared` | Galaxy publishing | `no-changed-when`, `no-handler`, `meta-no-tags`, `max-tasks` |
| `production` | AAP certified content | `fqcn`, `sanity`, `single-entry-point`, `meta-no-dependencies` |

```bash
# Run with a specific profile
ansible-lint --profile=shared roles/

# Check which profile your content currently passes
ansible-lint --profile=production playbook.yml 2>&1 | grep "profile:"
```

## Configuration File

ansible-lint reads config from `.ansible-lint` in the project root (YAML format):

```yaml
# .ansible-lint
profile: shared

# Specific rules to skip
skip_list:
  - name[casing]
  - yaml[line-length]

# Rules that produce warnings instead of errors
warn_list:
  - no-changed-when
  - risky-file-permissions

# Paths to exclude from linting
exclude_paths:
  - .git/
  - .cache/
  - molecule/
  - tests/

# Enable optional rules
enable_list:
  - no-same-owner

# Offline mode (don't download dependencies)
offline: false

# Use progressive mode (only report new violations)
progressive: true
```

> **Progressive mode:** Only reports violations introduced since the last clean run. Useful for large existing codebases where fixing everything at once isn't practical.

## Common Rules and Fixes

### command-instead-of-module

```yaml
# BAD — using shell to install packages
- name: Install nginx
  ansible.builtin.shell: yum install -y nginx

# GOOD — use the dedicated module
- name: Install nginx
  ansible.builtin.yum:
    name: nginx
    state: present
```

### fqcn[action-core]

```yaml
# BAD — short module name
- name: Copy config
  copy:
    src: app.conf
    dest: /etc/app/app.conf

# GOOD — fully qualified collection name
- name: Copy config
  ansible.builtin.copy:
    src: app.conf
    dest: /etc/app/app.conf
```

### no-changed-when

```yaml
# BAD — command always reports "changed"
- name: Check disk space
  ansible.builtin.command: df -h /
  register: disk_output

# GOOD — tell Ansible when it actually changed something
- name: Check disk space
  ansible.builtin.command: df -h /
  register: disk_output
  changed_when: false

# GOOD — conditional changed_when
- name: Run database migration
  ansible.builtin.command: /opt/app/migrate.sh
  register: migrate_result
  changed_when: "'Applied' in migrate_result.stdout"
```

### name[play] and name[casing]

```yaml
# BAD — no play name, task name starts lowercase
---
- hosts: webservers
  tasks:
    - name: install package
      ansible.builtin.apt:
        name: nginx
        state: present

# GOOD — named play, uppercase first letter
---
- name: Configure web servers
  hosts: webservers
  tasks:
    - name: Install nginx package
      ansible.builtin.apt:
        name: nginx
        state: present
```

### risky-file-permissions

```yaml
# BAD — no mode specified (inherits umask, unpredictable)
- name: Create config file
  ansible.builtin.copy:
    src: app.conf
    dest: /etc/app/app.conf

# GOOD — explicit permissions
- name: Create config file
  ansible.builtin.copy:
    src: app.conf
    dest: /etc/app/app.conf
    mode: "0644"
```

### package-latest

```yaml
# BAD — unpredictable (different version on each run)
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: latest

# GOOD — idempotent, predictable
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
```

### yaml[line-length]

```yaml
# Default max is 160 characters
# To adjust, add to .ansible-lint or .yamllint:

# .yamllint
---
rules:
  line-length:
    max: 200
    allow-non-breakable-words: true
    allow-non-breakable-inline-mappings: true
```

### no-free-form

```yaml
# BAD — free-form arguments (error-prone with special characters)
- name: Run script
  ansible.builtin.command: /opt/scripts/deploy.sh --env production

# GOOD — use cmd or argv
- name: Run script
  ansible.builtin.command:
    cmd: /opt/scripts/deploy.sh --env production
```

## Inline Rule Skipping

Skip a specific rule for a single task using `# noqa`:

```yaml
# Skip a specific rule on one task
- name: Run legacy script  # noqa: command-instead-of-module
  ansible.builtin.shell: /opt/legacy/run.sh

# Skip multiple rules
- name: Quick fix  # noqa: no-changed-when risky-shell-pipe
  ansible.builtin.shell: cat /tmp/data | grep error
  register: errors
```

> Use `# noqa` sparingly. If you're skipping rules frequently, you might want to adjust your profile or fix the underlying pattern.

## CI/CD Integration

### GitHub Actions

```yaml
name: Ansible Lint
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install ansible-lint
        run: pip3 install ansible-lint

      - name: Run ansible-lint
        run: ansible-lint .
```

### GitLab CI

```yaml
ansible-lint:
  image: python:3.11-slim
  stage: validate
  script:
    - pip install ansible-lint
    - ansible-lint .
  rules:
    - changes:
        - "**/*.yml"
        - "**/*.yaml"
        - roles/**/*
```

### Pre-commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/ansible/ansible-lint
    rev: v24.7.0
    hooks:
      - id: ansible-lint
        additional_dependencies:
          - ansible-core>=2.15
```

```bash
# Install and run
pip install pre-commit
pre-commit install
pre-commit run ansible-lint --all-files
```

## Before and After Example

A poorly written playbook and its corrected version:

```yaml
# BEFORE — 5 violations
---
- hosts: localhost
  tasks:
    - name: install package
      shell: |
        yum install -y sos
```

```bash
$ ansible-lint test.yml

WARNING  Listing 5 violation(s) that are fatal

name[play]: All plays should be named.
test.yml:2

command-instead-of-module: yum used in place of yum module
test.yml:4 Task/Handler: install package

fqcn[action-core]: Use FQCN for builtin module actions (shell).
test.yml:4 Use `ansible.builtin.shell` or `ansible.legacy.shell` instead.

name[casing]: All names should start with an uppercase letter.
test.yml:4 Task/Handler: install package

no-changed-when: Commands should not change things if nothing needs doing.
test.yml:4 Task/Handler: install package

Failed after min profile: 5 failure(s), 0 warning(s) on 1 files.
```

```yaml
# AFTER — passes production profile
---
- name: Install sos on servers
  hosts: localhost
  tasks:
    - name: Install sos package
      ansible.builtin.yum:
        name: sos
        state: present
```

```bash
$ ansible-lint test.yml
Passed with production profile: 0 failure(s), 0 warning(s) on 1 files.
```

## Useful Commands

```bash
# Lint with verbose output (shows which rules are checked)
ansible-lint -v playbook.yml

# Show config being used
ansible-lint --show-relpath

# Generate a default config file
ansible-lint --generate-config > .ansible-lint

# List all tags
ansible-lint -T

# Run only specific rules
ansible-lint -R -r /path/to/custom/rules playbook.yml

# Format output as JSON (useful for CI parsing)
ansible-lint --format json playbook.yml

# Format as SARIF (GitHub code scanning)
ansible-lint --format sarif playbook.yml > results.sarif

# Check what version/profile your project passes
ansible-lint --profile=min . && echo "passes min"
ansible-lint --profile=basic . && echo "passes basic"
ansible-lint --profile=shared . && echo "passes shared"
ansible-lint --profile=production . && echo "passes production"
```

Source: [ansible-lint documentation](https://ansible.readthedocs.io/projects/lint/)
