# Ansible Ad-Hoc Commands vs Playbooks

Why ad-hoc commands feel faster, when playbooks are the better choice, and how to decide which approach fits the job.

## Why Ad-Hoc Commands Are Faster

Ad-hoc commands skip the overhead that playbooks carry. The speed difference comes from several factors:

| Factor | Ad-Hoc | Playbook |
|--------|--------|----------|
| YAML parsing | None | Parses full YAML structure |
| Task compilation | Single task | Compiles all tasks, handlers, roles |
| Variable resolution | Minimal | Full scope (group_vars, host_vars, facts) |
| Fact gathering | Skipped by default | Runs `setup` module first (adds 2–5s per host) |
| Callback plugins | Minimal output | Full callback processing |
| Role/include loading | None | Resolves all dependencies |
| Handler notification | None | Tracks and flushes handlers |

### Fact Gathering Is the Biggest Overhead

By default, playbooks run `gather_facts: true` which calls the `setup` module on every host before any task runs. This alone adds 2–5 seconds per host.

```bash
# Ad-hoc: immediate execution, no facts
time ansible webservers -m command -a "uptime"
# Real: ~0.8s for 5 hosts

# Playbook equivalent (with default fact gathering)
time ansible-playbook check_uptime.yml
# Real: ~4.2s for 5 hosts
```

You can disable it in playbooks with `gather_facts: false`, but then you lose access to `ansible_*` variables.

### Startup Comparison

```bash
# Ad-hoc: straight to execution
ansible all -m ping

# Playbook: parse → compile → variable resolution → fact gather → execute
ansible-playbook site.yml
```

Even an empty playbook with a single `ping` task is measurably slower than `ansible all -m ping` because of the YAML parsing and task compilation overhead.

## When Ad-Hoc Commands Win

### Quick Checks and Troubleshooting

```bash
# Check disk space across all servers
ansible all -m command -a "df -h /"

# See who's logged in
ansible webservers -m command -a "w"

# Check a service status
ansible dbservers -m command -a "systemctl status postgresql"

# Verify connectivity
ansible all -m ping
```

### One-Off Operations

```bash
# Restart a service after a config change you made manually
ansible webservers -m systemd -a "name=nginx state=restarted" -b

# Kill a runaway process
ansible app01 -m command -a "pkill -f 'zombie-process'" -b

# Copy an emergency hotfix
ansible webservers -m copy -a "src=hotfix.conf dest=/etc/myapp/hotfix.conf" -b
```

### Gathering Information

```bash
# Check OS version across fleet
ansible all -m command -a "cat /etc/os-release" -o

# List installed kernel versions
ansible all -m command -a "rpm -q kernel" -o

# Check memory on specific hosts
ansible 'web01,web02,web03' -m command -a "free -h"
```

### File Operations

```bash
# Create a directory
ansible all -m file -a "path=/opt/backups state=directory mode=0755" -b

# Remove a temp file from all servers
ansible all -m file -a "path=/tmp/stale.lock state=absent" -b

# Change ownership
ansible all -m file -a "path=/var/log/myapp owner=appuser group=appuser recurse=yes" -b
```

## When Playbooks Win

### Idempotency and State Management

Playbooks track state across multiple tasks. Ad-hoc commands are fire-and-forget — you have no way to express "only do X if Y is true" in a single command.

```yaml
# Playbook: conditional logic, handlers, state tracking
- hosts: webservers
  become: true
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Deploy config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: restart nginx

    - name: Ensure running
      systemd:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
```

Doing this with ad-hoc commands would be 4+ separate invocations with no conditional linking between them.

### Multi-Step Workflows

```yaml
# Playbook: rolling update with pre/post checks
- hosts: webservers
  serial: 2
  become: true
  tasks:
    - name: Remove from load balancer
      uri:
        url: "http://lb.internal/api/drain/{{ inventory_hostname }}"
        method: POST

    - name: Update application
      apt:
        name: myapp
        state: latest

    - name: Run migrations
      command: /opt/myapp/bin/migrate
      run_once: true

    - name: Health check
      uri:
        url: "http://{{ inventory_hostname }}:8080/health"
        status_code: 200
      retries: 5
      delay: 3

    - name: Re-add to load balancer
      uri:
        url: "http://lb.internal/api/enable/{{ inventory_hostname }}"
        method: POST
```

No ad-hoc equivalent exists for serial execution, retries, or run_once logic.

### Repeatability and Version Control

Playbooks live in git. You can review changes, revert mistakes, and run the exact same operation next month. Ad-hoc commands live in your bash history (if you're lucky).

### Roles and Reuse

```bash
# Ad-hoc: repeat the same 10 commands for every new server
ansible newhost -m apt -a "name=curl state=present" -b
ansible newhost -m apt -a "name=vim state=present" -b
ansible newhost -m user -a "name=deploy shell=/bin/bash" -b
# ... and 7 more commands

# Playbook: one line
# ansible-playbook bootstrap.yml -l newhost
```

## Side-by-Side Comparison

| Criteria | Ad-Hoc | Playbook |
|----------|--------|----------|
| Speed (single task) | Faster | Slower (overhead) |
| Multi-task workflows | Manual chaining | Native support |
| Idempotency | Module-level only | Full task chain |
| Error handling | None | `block/rescue/always` |
| Conditionals | None | `when`, `failed_when`, `changed_when` |
| Loops | None | `loop`, `with_items` |
| Templates (Jinja2) | Not supported | Full support |
| Handlers | Not available | Notify/flush |
| Dry run | `--check` (limited) | `--check` + `--diff` |
| Rolling updates | Not possible | `serial`, `max_fail_percentage` |
| Vault secrets | `--ask-vault-pass` only | Full vault integration |
| Reusability | None (one-shot) | Roles, includes, imports |
| Auditability | Bash history | Git history |
| Documentation | None | Self-documenting YAML |
| CI/CD integration | Awkward | Native |

## Performance Tips for Playbooks

If playbook overhead bothers you, these settings narrow the gap:

### Disable Fact Gathering When Not Needed

```yaml
- hosts: all
  gather_facts: false
  tasks:
    - name: Quick check
      command: uptime
```

### Use Fact Caching

```ini
# ansible.cfg
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600
```

### Increase Parallelism

```ini
# ansible.cfg
[defaults]
forks = 20

[ssh_connection]
pipelining = true
```

### Use Mitogen (Strategy Plugin)

Mitogen eliminates most SSH overhead by reusing connections and transferring Python code directly:

```ini
# ansible.cfg
[defaults]
strategy_plugins = /path/to/mitogen/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
```

## Decision Flowchart

Use this to decide which approach fits:

```
Need to do it once, right now?
├── Yes → Is it a single module call?
│         ├── Yes → Ad-hoc ✓
│         └── No (multiple steps) → Quick script or playbook
└── No (repeatable) → Playbook ✓

Need conditionals, loops, or handlers?
└── Yes → Playbook ✓

Need it in CI/CD or scheduled?
└── Yes → Playbook ✓

Debugging / checking state across hosts?
└── Yes → Ad-hoc ✓
```

## Common Mistakes

- **Using ad-hoc for config management** — One missed command and your fleet drifts. Playbooks enforce the full desired state every run.
- **Avoiding ad-hoc entirely** — Writing a playbook to check disk space is overkill. Ad-hoc is a legitimate operational tool.
- **Forgetting `-b` (become)** — Ad-hoc commands don't inherit `become: true` from playbook defaults. You must pass `-b` explicitly.
- **Not using `--check` before ad-hoc changes** — There's no undo. Run with `--check` first on destructive operations.
- **Chaining ad-hoc commands in scripts** — If your bash script has 5+ `ansible` calls in sequence, it should be a playbook.
