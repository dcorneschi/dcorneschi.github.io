# Ansible Cheatsheet

Ansible is an agentless automation tool that uses SSH to configure systems, deploy software, and orchestrate workflows. It uses YAML playbooks to describe desired state and executes tasks against inventory hosts in parallel.

## Install

```bash
# pip (recommended — latest version)
pip3 install ansible

# Ubuntu / Debian
sudo apt update && sudo apt install ansible

# RHEL / Fedora
sudo dnf install ansible-core

# CentOS (requires EPEL)
sudo yum install epel-release
sudo yum install ansible

# macOS
brew install ansible

# Verify
ansible --version
ansible-playbook --version

# Install Python JSON library on managed nodes (required for very old systems)
# Only needed if target has Python 2.4 without json module
ansible all -m raw -a "yum install -y python-simplejson" -b
```

## SSH Setup for Ansible

```bash
# Generate SSH key pair (if not already done)
ssh-keygen -t rsa -b 4096

# Copy public key to managed hosts (enables passwordless SSH)
ssh-copy-id user@hostname
ssh-copy-id -i ~/.ssh/id_rsa.pub user@192.168.1.10

# Test connection
ssh user@hostname

# Pre-populate known_hosts for all inventory hosts
for i in $(cat inventory.txt); do
  ssh-keyscan "$i" >> ~/.ssh/known_hosts
done
```

## Disable Host Key Checking

```bash
# Temporary (current shell session only)
export ANSIBLE_HOST_KEY_CHECKING=False

# Permanent (in ansible.cfg)
# [defaults]
# host_key_checking = False

# Permanent (in ~/.ansible.cfg)
cat <<'EOF' > ~/.ansible.cfg
[defaults]
host_key_checking = False
EOF

# Per-command
ANSIBLE_HOST_KEY_CHECKING=False ansible all -m ping
```

### Pre-populate known_hosts (Preferred Over Disabling)

```bash
# Add specific hosts
ssh-keyscan server1.example.com >> ~/.ssh/known_hosts
ssh-keyscan server2.example.com >> ~/.ssh/known_hosts

# Add all hosts from inventory file
for i in $(cat inventory.txt); do
  ssh-keyscan "$i" >> ~/.ssh/known_hosts
done

# Add all hosts from Ansible inventory
ansible all --list-hosts | tail -n +2 | xargs -I {} ssh-keyscan {} >> ~/.ssh/known_hosts
```

> **Security note:** Disabling host key checking skips SSH MITM protection. In production, pre-populate `known_hosts` with `ssh-keyscan` instead.

## Core Commands

| Command | Purpose |
|---------|---------|
| `ansible` | Run ad-hoc commands on hosts |
| `ansible-playbook` | Execute playbooks |
| `ansible-inventory` | Show/manage inventory |
| `ansible-vault` | Encrypt/decrypt sensitive data |
| `ansible-galaxy` | Install roles and collections |
| `ansible-config` | View/dump configuration |
| `ansible-doc` | Module documentation |
| `ansible-pull` | Pull and run playbooks from VCS |
| `ansible-lint` | Lint playbooks for best practices |

## ansible-playbook

```bash
# Run a playbook
ansible-playbook playbook.yml

# Specify inventory
ansible-playbook -i inventory.ini playbook.yml
ansible-playbook -i production/ playbook.yml

# Dry run (check mode)
ansible-playbook playbook.yml --check
ansible-playbook playbook.yml --check --diff

# Limit to specific hosts
ansible-playbook playbook.yml --limit web01
ansible-playbook playbook.yml --limit webservers
ansible-playbook playbook.yml --limit 'webservers:!web03'

# Pass extra variables
ansible-playbook playbook.yml -e "env=production version=2.1"
ansible-playbook playbook.yml -e "@vars.yml"
ansible-playbook playbook.yml --extra-vars '{"env": "prod", "debug": true}'

# Run with tags
ansible-playbook playbook.yml --tags "deploy,restart"
ansible-playbook playbook.yml --skip-tags "debug"

# List tasks/hosts/tags without executing
ansible-playbook playbook.yml --list-tasks
ansible-playbook playbook.yml --list-hosts
ansible-playbook playbook.yml --list-tags

# Start at a specific task
ansible-playbook playbook.yml --start-at-task="Install nginx"

# Step through tasks (confirm each)
ansible-playbook playbook.yml --step

# Verbose output
ansible-playbook playbook.yml -v
ansible-playbook playbook.yml -vvv

# Set forks (parallel hosts)
ansible-playbook playbook.yml -f 20

# Become (sudo)
ansible-playbook playbook.yml -b
ansible-playbook playbook.yml --become --become-user=root --ask-become-pass

# Syntax check (no execution)
ansible-playbook playbook.yml --syntax-check

# Flush handlers and finish early
ansible-playbook playbook.yml --flush-cache

# Use vault password
ansible-playbook playbook.yml --ask-vault-pass
ansible-playbook playbook.yml --vault-password-file=~/.vault_pass
```

## Inventory

### INI Format

```ini
# inventory.ini
[webservers]
web01 ansible_host=192.168.1.10
web02 ansible_host=192.168.1.11
web03 ansible_host=192.168.1.12

[databases]
db01 ansible_host=192.168.1.20 ansible_port=2222
db02 ansible_host=192.168.1.21

[loadbalancers]
lb01 ansible_host=192.168.1.5

# Group of groups
[production:children]
webservers
databases
loadbalancers

# Group variables
[webservers:vars]
ansible_user=deploy
ansible_python_interpreter=/usr/bin/python3
http_port=80

[all:vars]
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### YAML Format

```yaml
# inventory.yml
all:
  vars:
    ansible_user: deploy
    ansible_python_interpreter: /usr/bin/python3
  children:
    webservers:
      hosts:
        web01:
          ansible_host: 192.168.1.10
        web02:
          ansible_host: 192.168.1.11
      vars:
        http_port: 80
    databases:
      hosts:
        db01:
          ansible_host: 192.168.1.20
        db02:
          ansible_host: 192.168.1.21
    production:
      children:
        webservers:
        databases:
```

### Inventory Commands

```bash
# List all hosts
ansible-inventory --list
ansible-inventory --graph
ansible-inventory --graph --vars

# Show specific host vars
ansible-inventory --host web01

# List all groups
ansible localhost -m debug -a 'var=groups.keys()'

# Use multiple inventories
ansible-playbook -i staging/ -i production/ playbook.yml
```

### Dynamic Grouping with group_by

Create host groups at runtime based on facts — no need to hardcode OS-specific groups in inventory:

```yaml
# First play: classify hosts by OS
- name: Group hosts by OS
  hosts: all
  tasks:
    - name: Create dynamic groups based on distribution
      group_by:
        key: "os_{{ ansible_distribution }}"

# Second play: target only Ubuntu hosts
- name: Configure Ubuntu servers
  hosts: os_Ubuntu
  tasks:
    - name: Install packages via apt
      apt:
        name: nginx
        state: present

# Third play: target only RedHat-family hosts
- name: Configure RHEL servers
  hosts: os_RedHat
  tasks:
    - name: Install packages via dnf
      dnf:
        name: nginx
        state: present
```

> `group_by` runs during the play and creates ephemeral groups that exist only for the current playbook run. Useful when the same inventory contains mixed OS hosts and you want different tasks per OS without duplicating inventory groups.

### Common Host Variables

| Variable | Description |
|----------|-------------|
| `ansible_host` | IP or hostname to connect to |
| `ansible_port` | SSH port (default: 22) |
| `ansible_user` | SSH user |
| `ansible_password` | SSH password (use vault!) |
| `ansible_ssh_private_key_file` | SSH key path |
| `ansible_python_interpreter` | Path to Python on target |
| `ansible_become` | Enable privilege escalation |
| `ansible_become_user` | Escalation user |
| `ansible_become_method` | Method (sudo, su, pbrun, etc.) |
| `ansible_connection` | Connection type (ssh, local, docker) |
| `ansible_shell_type` | Shell type (sh, csh, fish) |

## Playbook Structure

### Minimal Playbook

```yaml
---
- name: Install and start nginx
  hosts: webservers
  become: yes

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
```

### Full Playbook

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: yes
  gather_facts: yes
  vars:
    http_port: 80
    app_version: "2.1.0"
  vars_files:
    - vars/common.yml
    - vars/{{ env }}.yml

  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

  roles:
    - common
    - nginx
    - app

  tasks:
    - name: Deploy application
      copy:
        src: "app-{{ app_version }}.tar.gz"
        dest: /opt/app/
      notify: restart app

  post_tasks:
    - name: Verify deployment
      uri:
        url: "http://localhost:{{ http_port }}/health"
        status_code: 200

  handlers:
    - name: restart app
      service:
        name: myapp
        state: restarted
```

### Task Keywords

```yaml
- name: Example task with common keywords
  apt:
    name: nginx
    state: present
  become: yes
  become_user: root
  when: ansible_os_family == "Debian"
  register: install_result
  changed_when: install_result.changed
  failed_when: install_result.rc != 0
  ignore_errors: yes
  retries: 3
  delay: 5
  until: install_result is success
  notify: restart nginx
  tags:
    - packages
    - nginx
  environment:
    HTTP_PROXY: "http://proxy:3128"
```

## Variables

### Variable Precedence (Low → High)

1. Role defaults (`roles/x/defaults/main.yml`)
2. Inventory group_vars (`group_vars/all`)
3. Inventory group_vars (`group_vars/groupname`)
4. Inventory host_vars (`host_vars/hostname`)
5. Playbook group_vars
6. Playbook host_vars
7. Host facts (`ansible_*`)
8. Play vars
9. Play vars_prompt
10. Play vars_files
11. Role vars (`roles/x/vars/main.yml`)
12. Block vars
13. Task vars
14. Include_vars
15. Set_facts / registered vars
16. Extra vars (`-e`) — **always win**

### Variable Definition

```yaml
# In playbook
vars:
  app_name: myapp
  app_port: 8080
  packages:
    - nginx
    - git
    - curl

# From file
vars_files:
  - vars/common.yml

# Prompted at runtime
vars_prompt:
  - name: deploy_version
    prompt: "Version to deploy?"
    private: no
```

### Directory Structure for Variables

```
inventory/
├── group_vars/
│   ├── all.yml            # All hosts
│   ├── webservers.yml     # webservers group
│   └── production.yml     # production group
├── host_vars/
│   ├── web01.yml          # web01 only
│   └── db01.yml           # db01 only
└── hosts.ini
```

### Using Variables

```yaml
# Simple
"{{ app_name }}"

# Default value
"{{ app_port | default(8080) }}"

# Dictionary access
"{{ db_config.host }}"
"{{ db_config['host'] }}"

# List access
"{{ servers[0] }}"

# Registered variables
- name: Check service
  command: systemctl status nginx
  register: nginx_status

- name: Print result
  debug:
    msg: "{{ nginx_status.stdout }}"

# Set fact (create variable at runtime)
- name: Set derived variable
  set_fact:
    full_app_name: "{{ app_name }}-{{ app_version }}"

- name: Use set_fact
  debug:
    msg: "Deploying {{ full_app_name }}"
```

## Conditionals

```yaml
# Simple when
- name: Install on Debian
  apt:
    name: nginx
  when: ansible_os_family == "Debian"

# Multiple conditions
- name: Install on Ubuntu 22.04+
  apt:
    name: nginx
  when:
    - ansible_distribution == "Ubuntu"
    - ansible_distribution_major_version | int >= 22

# Or conditions
- name: Install on Debian or Ubuntu
  apt:
    name: nginx
  when: ansible_distribution == "Debian" or ansible_distribution == "Ubuntu"

# Variable defined
- name: Only if var exists
  debug:
    msg: "{{ custom_var }}"
  when: custom_var is defined

# Boolean
- name: Only in production
  service:
    name: monitoring
    state: started
  when: enable_monitoring | bool

# Register + when
- name: Check if file exists
  stat:
    path: /opt/app/config.yml
  register: config_file

- name: Create config if missing
  template:
    src: config.yml.j2
    dest: /opt/app/config.yml
  when: not config_file.stat.exists
```

## Loops

```yaml
# Simple list
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - git
    - curl

# Better: pass list directly (faster)
- name: Install packages
  apt:
    name:
      - nginx
      - git
      - curl
    state: present

# Loop with dict
- name: Create users
  user:
    name: "{{ item.name }}"
    uid: "{{ item.uid }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: alice, uid: 1001, groups: "sudo" }
    - { name: bob, uid: 1002, groups: "docker" }

# Loop with index
- name: Create numbered files
  file:
    path: "/tmp/file-{{ idx }}"
    state: touch
  loop: "{{ range(1, 6) | list }}"
  loop_control:
    loop_var: idx

# Nested loops
- name: Grant access
  mysql_user:
    name: "{{ item[0] }}"
    priv: "{{ item[1] }}.*:ALL"
  with_nested:
    - ['alice', 'bob']
    - ['db1', 'db2']

# Loop over dict
- name: Set sysctl values
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
  loop: "{{ sysctl_settings | dict2items }}"

# Legacy syntax (with_items — still works but loop is preferred)
- name: Install packages (legacy)
  yum:
    name: "{{ item }}"
    state: present
  with_items:
    - git
    - vim
    - curl
```

## Handlers

```yaml
# Handlers run once at end of play, only if notified
handlers:
  - name: restart nginx
    service:
      name: nginx
      state: restarted

  - name: reload nginx
    service:
      name: nginx
      state: reloaded

  - name: restart and enable nginx
    service:
      name: nginx
      state: restarted
      enabled: yes

# Notify from a task
tasks:
  - name: Update nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify:
      - reload nginx

# Flush handlers immediately (don't wait until end of play)
  - name: Flush handlers now
    meta: flush_handlers
```

## Roles

### Role Structure

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── nginx.conf.j2
    ├── files/
    │   └── index.html
    ├── vars/
    │   └── main.yml
    ├── defaults/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    └── README.md
```

### Using Roles

```yaml
# In playbook
- hosts: webservers
  roles:
    - common
    - nginx
    - { role: app, app_version: "2.1.0" }
    - role: monitoring
      when: enable_monitoring | bool
      tags: monitoring
```

### import_role / include_role (Inline Role Execution)

The `roles:` keyword runs all roles before any `tasks:`. To mix roles and tasks in a specific order, use `import_role` or `include_role` inside the tasks section:

```yaml
- name: Deploy with controlled ordering
  hosts: webservers
  become: yes
  tasks:
    - name: Pre-deployment health check
      uri:
        url: "http://{{ inventory_hostname }}/health"
        status_code: 200

    - name: Apply nginx configuration role
      import_role:
        name: nginx
      vars:
        nginx_port: 8080

    - name: Deploy application role
      include_role:
        name: app
      when: deploy_app | default(true)

    - name: Post-deployment verification
      uri:
        url: "http://{{ inventory_hostname }}:8080/health"
        status_code: 200
```

| | `import_role` | `include_role` |
|---|---|---|
| **Processing** | Static — parsed at playbook load time | Dynamic — processed at runtime |
| **Tags/when** | Applied to every task inside the role | Applied to the include statement only |
| **Loops** | Cannot be used in a loop | Can be used with `loop` |
| **Handlers** | Role handlers visible globally | Role handlers scoped to include |
| **Use when** | Default choice — predictable behavior | Need conditional or looped role inclusion |

> **Rule of thumb:** Use `import_role` unless you need to loop over it or conditionally skip entire roles at runtime. `import_role` is more predictable because all tasks are visible during `--list-tasks`.

### ansible-galaxy

```bash
# Install role from Galaxy
ansible-galaxy install geerlingguy.nginx
ansible-galaxy install geerlingguy.docker

# Install from requirements file
ansible-galaxy install -r requirements.yml

# Install collection
ansible-galaxy collection install community.general
ansible-galaxy collection install amazon.aws

# Install role from git URL
ansible-galaxy install git+https://github.com/org/role.git

# List installed
ansible-galaxy list
ansible-galaxy collection list

# Remove a role
ansible-galaxy remove role_name

# Init a new role
ansible-galaxy init my_role

# Init a new collection
ansible-galaxy collection init my_namespace.my_collection
```

### requirements.yml

```yaml
---
roles:
  - name: geerlingguy.nginx
    version: "3.1.0"
  - name: geerlingguy.docker
  - src: https://github.com/org/role.git
    scm: git
    version: main
    name: custom_role

collections:
  - name: community.general
    version: ">=7.0.0"
  - name: amazon.aws
    version: "6.5.0"
```

## Ansible Vault

```bash
# Create encrypted file
ansible-vault create secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Encrypt existing file
ansible-vault encrypt vars/production.yml

# Decrypt file
ansible-vault decrypt vars/production.yml

# View encrypted file (don't edit)
ansible-vault view secrets.yml

# Change vault password
ansible-vault rekey secrets.yml

# Encrypt a string (inline)
ansible-vault encrypt_string 'my_secret_password' --name 'db_password'

# Run playbook with vault
ansible-playbook playbook.yml --ask-vault-pass
ansible-playbook playbook.yml --vault-password-file=~/.vault_pass
ansible-playbook playbook.yml --vault-id prod@~/.vault_pass_prod
```

### Using Vault in Playbooks

```yaml
# vars/secrets.yml (encrypted)
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  3862346538663766326234...

# Reference normally
- name: Configure database
  template:
    src: db.conf.j2
    dest: /etc/app/db.conf
  vars:
    password: "{{ db_password }}"
```

## Templates (Jinja2)

```jinja
{# templates/nginx.conf.j2 #}

server {
    listen {{ http_port | default(80) }};
    server_name {{ ansible_hostname }};

    root /var/www/{{ app_name }};

{% if enable_ssl | default(false) %}
    listen 443 ssl;
    ssl_certificate /etc/ssl/{{ domain }}.crt;
    ssl_certificate_key /etc/ssl/{{ domain }}.key;
{% endif %}

{% for upstream in app_servers %}
    upstream backend {
        server {{ upstream.host }}:{{ upstream.port }};
    }
{% endfor %}

    location / {
        proxy_pass http://backend;
    }
}
```

### Common Filters

```yaml
# String
"{{ name | upper }}"
"{{ name | lower }}"
"{{ name | capitalize }}"
"{{ path | basename }}"
"{{ path | dirname }}"
"{{ text | regex_replace('old', 'new') }}"

# Default value
"{{ var | default('fallback') }}"
"{{ var | default(omit) }}"

# Type conversion
"{{ port | int }}"
"{{ flag | bool }}"
"{{ data | to_json }}"
"{{ data | to_yaml }}"
"{{ data | from_json }}"

# List
"{{ list | join(', ') }}"
"{{ list | first }}"
"{{ list | last }}"
"{{ list | length }}"
"{{ list | unique }}"
"{{ list | sort }}"
"{{ list | flatten }}"

# Hash/Password
"{{ 'secret' | password_hash('sha512') }}"

# File
"{{ lookup('file', '/path/to/file') }}"
"{{ lookup('env', 'HOME') }}"
"{{ lookup('template', 'template.j2') }}"

# IP address
"{{ ansible_default_ipv4.address }}"
"{{ '192.168.1.0/24' | ipaddr('network') }}"
```

## Configuration (ansible.cfg)

```ini
# ansible.cfg (in project directory — highest priority)
[defaults]
inventory = ./inventory
remote_user = deploy
private_key_file = ~/.ssh/id_rsa
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 86400
stdout_callback = yaml
forks = 20
timeout = 30
log_path = /var/log/ansible.log
bin_ansible_callbacks = True

[inventory]
enable_plugins = host_list, script, auto, yaml, ini, toml

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
control_path_dir = /tmp/.ansible-cp
```

### Performance Tuning

```yaml
# In playbook — optimize execution
---
- name: Optimized playbook
  hosts: all
  gather_facts: no         # Skip if facts not needed
  strategy: free           # Don't wait for slowest host before proceeding
  serial: 10              # Limit concurrent hosts

  tasks:
    - name: Gather only needed facts
      setup:
        filter: ansible_distribution*
      when: ansible_facts is not defined
```

```ini
# In ansible.cfg — fact caching (avoid re-gathering)
[defaults]
gathering = smart                    # Only gather facts if not cached
fact_caching = memory                # Cache in memory (or jsonfile/redis)
fact_caching_timeout = 86400         # Cache TTL in seconds (24h)
```

| Strategy | Behavior |
|----------|----------|
| `linear` (default) | Wait for all hosts to finish each task before moving to next |
| `free` | Each host runs as fast as it can without waiting for others |
| `debug` | Interactive debugger on task failure |

### Config Precedence (High → Low)

1. `ANSIBLE_CONFIG` environment variable (path to a config file)
2. `./ansible.cfg` (current directory)
3. `~/.ansible.cfg` (home directory — note the dot prefix)
4. `/etc/ansible/ansible.cfg` (system-wide default)

```bash
# Show effective configuration
ansible-config dump
ansible-config dump --only-changed
ansible-config list

# Show which config file is being used
ansible --version | grep "config file"
```

## System-Wide Paths

| Path | Purpose |
|------|---------|
| `/etc/ansible/` | Main system-wide configuration directory |
| `/etc/ansible/ansible.cfg` | System-wide default configuration |
| `/etc/ansible/hosts` | Default inventory file (used when no `-i` specified) |
| `/etc/ansible/roles/` | System-wide roles directory |

> **Note:** For project-based work, keep `ansible.cfg` and `inventory/` in your project directory rather than using the system-wide paths.

## Role Auto-Discovery

Within each role subdirectory (`tasks/`, `handlers/`, `templates/`, `files/`, `vars/`, `defaults/`, `meta/`), Ansible automatically searches for a `main.yml` file. You don't need to specify the filename explicitly.

```
roles/nginx/
├── tasks/main.yml        ← Ansible loads this automatically
├── handlers/main.yml     ← Ansible loads this automatically
├── templates/            ← Files referenced in template module
├── files/                ← Static files referenced in copy module
├── vars/main.yml         ← High-priority variables
├── defaults/main.yml     ← Low-priority defaults
└── meta/main.yml         ← Dependencies and metadata
```

| Directory | Purpose |
|-----------|---------|
| `tasks/` | The main list of tasks the role executes |
| `handlers/` | Handlers triggered by `notify` in tasks |
| `templates/` | Jinja2 templates (dynamic config files) |
| `files/` | Static files copied as-is to remote hosts |
| `vars/` | High-priority variables for the role |
| `defaults/` | Default variables (lowest priority — easy to override) |
| `meta/` | Role metadata — dependencies, galaxy info |

## ansible-doc

```bash
# List all modules
ansible-doc -l

# Show module documentation
ansible-doc apt
ansible-doc yum
ansible-doc copy
ansible-doc template

# Show module examples only
ansible-doc -s apt

# List modules matching pattern
ansible-doc -l | grep -i docker

# Show connection plugins
ansible-doc -t connection -l

# Show callback plugins
ansible-doc -t callback -l
```

## Common Patterns

### Deploy Application

```yaml
---
- name: Deploy application
  hosts: webservers
  become: yes
  serial: "25%"           # Rolling update — 25% at a time
  max_fail_percentage: 10  # Fail if >10% of hosts fail

  pre_tasks:
    - name: Disable in load balancer
      uri:
        url: "http://lb/api/disable/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost

  tasks:
    - name: Pull latest code
      git:
        repo: "{{ repo_url }}"
        dest: /opt/app
        version: "{{ app_version }}"
      notify: restart app

    - name: Install dependencies
      pip:
        requirements: /opt/app/requirements.txt
        virtualenv: /opt/app/venv

  post_tasks:
    - name: Re-enable in load balancer
      uri:
        url: "http://lb/api/enable/{{ inventory_hostname }}"
        method: POST
      delegate_to: localhost

  handlers:
    - name: restart app
      service:
        name: myapp
        state: restarted
```

### Error Handling

```yaml
# Block / Rescue / Always
- name: Handle errors
  block:
    - name: Attempt upgrade
      apt:
        upgrade: dist
      register: upgrade_result

    - name: Reboot if needed
      reboot:
      when: upgrade_result.changed

  rescue:
    - name: Rollback on failure
      command: /opt/scripts/rollback.sh

    - name: Notify admin
      mail:
        to: admin@example.com
        subject: "Deploy failed on {{ inventory_hostname }}"

  always:
    - name: Re-enable monitoring
      service:
        name: monitoring
        state: started
```

### Fail and Custom Failure Conditions

```yaml
# Fail explicitly with a message
- name: Fail if unsupported OS
  fail:
    msg: "This playbook only supports RedHat family"
  when: ansible_os_family != "RedHat"

# Ignore errors on a task
- name: Command that might fail
  command: /bin/false
  ignore_errors: yes

# Custom failure condition (fail only on specific exit codes)
- name: Run risky command
  command: /opt/risky-script.sh
  register: result
  failed_when: result.rc != 0 and result.rc != 2

# Custom changed condition
- name: Check configuration
  command: /opt/check-config.sh
  register: check_result
  changed_when: "'CHANGED' in check_result.stdout"
```

### Debug and Pause

```yaml
# Debug — show variable value
- name: Debug variable
  debug:
    var: my_variable

# Debug — show message
- name: Debug message
  debug:
    msg: "Host {{ inventory_hostname }} has IP {{ ansible_default_ipv4.address }}"

# Debug — only show in verbose mode (-vv or higher)
- name: Verbose-only debug
  debug:
    msg: "Detailed info for troubleshooting"
    verbosity: 2

# Pause — wait for user confirmation
- name: Pause before continuing
  pause:
    prompt: "Check the system before continuing. Press Enter to proceed."

# Pause — wait a specific time
- name: Wait 30 seconds
  pause:
    seconds: 30
```

### Delegation

```yaml
# Run task on a different host
- name: Add host to load balancer
  command: /usr/local/bin/lb-add {{ inventory_hostname }}
  delegate_to: lb01

# Run on localhost
- name: Wait for port
  wait_for:
    host: "{{ ansible_host }}"
    port: 80
    timeout: 60
  delegate_to: localhost
```

## Directory Layout (Best Practice)

```
ansible-project/
├── ansible.cfg
├── inventory/
│   ├── production/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   └── webservers.yml
│   │   └── host_vars/
│   │       └── web01.yml
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/
├── playbooks/
│   ├── site.yml              # Master playbook
│   ├── webservers.yml
│   └── databases.yml
├── roles/
│   ├── common/
│   ├── nginx/
│   └── app/
├── group_vars/               # Playbook-level group_vars
├── host_vars/                # Playbook-level host_vars
├── files/
├── templates/
├── vars/
│   ├── common.yml
│   └── secrets.yml           # Vault encrypted
├── requirements.yml          # Galaxy dependencies
└── Makefile                  # Convenience targets
```

## RHEL System Roles

Red Hat provides pre-built Ansible roles for common RHEL administration tasks. They're installed from the `rhel-system-roles` package.

### Install

```bash
# RHEL 9 / 8 (included in base repos)
sudo dnf install rhel-system-roles

# RHEL 8 (if ansible repo needed)
sudo subscription-manager repos --enable ansible-2-for-rhel-8-x86_64-rpms
sudo dnf install rhel-system-roles ansible

# RHEL 7
sudo subscription-manager repos --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-ansible-2-rpms
sudo yum install rhel-system-roles ansible
```

### Installed Locations

| Path | Content |
|------|---------|
| `/usr/share/ansible/roles/rhel-system-roles.*` | Role files |
| `/usr/share/doc/rhel-system-roles/` | Documentation per subsystem |

### Available Roles

| Role | Purpose |
|------|---------|
| `rhel-system-roles.timesync` | Configure NTP/chrony |
| `rhel-system-roles.network` | Configure networking |
| `rhel-system-roles.firewall` | Manage firewalld |
| `rhel-system-roles.selinux` | Configure SELinux |
| `rhel-system-roles.kdump` | Configure kdump |
| `rhel-system-roles.storage` | Manage disks, LVM, filesystems |
| `rhel-system-roles.logging` | Configure rsyslog |
| `rhel-system-roles.metrics` | Performance metrics (PCP) |
| `rhel-system-roles.tlog` | Session recording |
| `rhel-system-roles.crypto_policies` | System-wide crypto policies |
| `rhel-system-roles.certificate` | Manage certificates |
| `rhel-system-roles.nbde_client` | Network-Bound Disk Encryption |
| `rhel-system-roles.postfix` | Configure Postfix |

### Example Usage

```yaml
---
- name: Configure time synchronization
  hosts: all
  become: yes
  roles:
    - role: rhel-system-roles.timesync
      vars:
        timesync_ntp_servers:
          - hostname: ntp1.example.com
            iburst: yes
          - hostname: ntp2.example.com
            iburst: yes
```

## Vim Configuration for YAML

Tabs break YAML parsing. YAML only allows spaces for indentation — tabs are forbidden. Configure vim to automatically use spaces when editing Ansible playbooks:

```bash
# Add to ~/.vimrc
autocmd FileType yaml setlocal ai ts=2 sw=2 et
```

Or target YAML files explicitly:

```bash
# Add to ~/.vimrc
au BufNewFile,BufRead *.yaml,*.yml set et ts=2 sw=2
```

| Setting | Meaning |
|---------|---------|
| `et` | expandtab — insert spaces when Tab is pressed (never actual tabs) |
| `ts=2` | tabstop — display width of a tab character as 2 spaces |
| `sw=2` | shiftwidth — indent/dedent by 2 spaces with `<` and `>` commands |
| `ai` | autoindent — new lines inherit indentation from the previous line |

> **Why this matters:** A single tab character in a YAML file causes a parse error. Ansible will fail with cryptic messages like `Syntax Error while loading YAML`. Always use spaces — the vim settings above make Tab key produce spaces automatically.

## Debugging

```bash
# Syntax check
ansible-playbook playbook.yml --syntax-check

# Lint playbook
ansible-lint playbook.yml

# List tasks
ansible-playbook playbook.yml --list-tasks

# Dry run
ansible-playbook playbook.yml --check --diff

# Step through tasks
ansible-playbook playbook.yml --step

# Start at specific task
ansible-playbook playbook.yml --start-at-task="Deploy app"

# Verbose (SSH debugging at -vvvv)
ansible-playbook playbook.yml -vvvv

# Debug module in playbook
- name: Debug variable
  debug:
    var: my_variable

- name: Debug message
  debug:
    msg: "Host {{ inventory_hostname }} has IP {{ ansible_default_ipv4.address }}"
```

## One-Liners

```bash
# Check playbook syntax
ansible-playbook --syntax-check playbook.yml

# Alternative: validate YAML with Python (no Ansible needed)
python3 -c 'import yaml, sys; yaml.safe_load(sys.stdin)' < playbook.yml && echo "Valid YAML"

# Lint all playbooks
ansible-lint playbooks/*.yml

# Run with vault + limit + tags
ansible-playbook -i production site.yml --vault-password-file=~/.vault \
  --limit webservers --tags deploy

# Generate encrypted password
ansible all -m debug -a "msg={{ 'mypassword' | password_hash('sha512') }}" -l localhost

# Quick facts dump for one host
ansible web01 -m setup | tee facts-web01.json

# Set SELinux to permissive on all hosts
ansible all -m selinux -a "policy=targeted state=permissive" -b

# Find log files across all servers
ansible all -m find -a "paths=/var/log patterns='*.log' age=7d" -b

# List all installed collections
ansible-galaxy collection list

# Refresh facts cache
ansible all -m setup --tree /tmp/facts
```

## Quick Reference

```bash
# Run playbook
ansible-playbook playbook.yml

# Limit hosts
ansible-playbook playbook.yml -l webservers

# Extra vars
ansible-playbook playbook.yml -e "var=value"

# Check mode (dry run)
ansible-playbook playbook.yml -C -D

# Tags
ansible-playbook playbook.yml -t deploy
ansible-playbook playbook.yml --skip-tags debug

# Vault
ansible-vault create secrets.yml
ansible-vault edit secrets.yml
ansible-vault encrypt file.yml
ansible-playbook playbook.yml --ask-vault-pass

# Galaxy
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install community.general
ansible-galaxy init my_role

# Doc
ansible-doc module_name
ansible-doc -l | grep pattern

# Config
ansible-config dump --only-changed
```

## Best Practices

1. **Version control** — keep playbooks, roles, and inventory in Git
2. **Use ansible-vault** — never store passwords/secrets in plain text
3. **Test with `--check`** — dry-run before applying to production
4. **Meaningful names** — name every play and task descriptively
5. **Idempotent tasks** — running the playbook twice should produce no changes
6. **Use roles** — extract reusable logic into roles, share via Galaxy
7. **Document** — add comments explaining why, not just what
8. **Pin versions** — pin role and collection versions in requirements.yml
9. **Least privilege** — only use `become` on tasks that need it
10. **Separate environments** — use different inventory files/dirs for staging vs production
