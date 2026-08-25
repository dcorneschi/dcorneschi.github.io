# Ansible Roles Directory Structure

Visual breakdown of the standard Ansible role directory layout — what each directory and file does.

## Directory Tree

```
roles/
├── role_name/
│   ├── tasks/
│   │   └── main.yml          ← Main task file for role execution
│   ├── handlers/
│   │   └── main.yml          ← Handlers triggered by notify actions
│   ├── templates/
│   │   └── config.j2         ← Jinja2 templates for dynamic config generation
│   ├── files/
│   │   ├── file.txt          ← Static files to be copied to hosts
│   │   └── script.sh         ← Scripts for use with the script module
│   ├── vars/
│   │   └── main.yml          ← High-priority variables scoped to the role
│   ├── defaults/
│   │   └── main.yml          ← Default variables (lowest priority, easy to override)
│   ├── meta/
│   │   └── main.yml          ← Role dependencies and Galaxy metadata
│   ├── library/              ← Custom Ansible modules for this role
│   ├── module_utils/         ← Shared helper utilities for custom modules
│   └── plugins/              ← Plugins for extending Ansible (filters, lookups, etc.)
├── role_2/                    ← Same structure
└── role_3/                    ← Same structure
```

## What Each Directory Does

| Directory | Purpose | Auto-loaded? |
|-----------|---------|:---:|
| `tasks/` | The main list of tasks the role executes | Yes |
| `handlers/` | Event-driven tasks triggered by `notify` | Yes |
| `templates/` | Jinja2 templates referenced by the `template` module | No (referenced) |
| `files/` | Static files referenced by `copy`, `script`, or `unarchive` modules | No (referenced) |
| `vars/` | High-priority variables — hard to override from outside the role | Yes |
| `defaults/` | Default variables — lowest priority, easily overridden by inventory, playbook, or CLI | Yes |
| `meta/` | Role metadata: dependencies, Galaxy info, supported platforms | Yes |
| `library/` | Custom modules available only when this role is used | Yes |
| `module_utils/` | Python utilities shared between custom modules in `library/` | Yes |
| `plugins/` | Custom plugins (filter, lookup, callback, etc.) scoped to this role | Yes |

> **Auto-loaded** means Ansible reads `main.yml` from that directory automatically when the role is applied. For `templates/` and `files/`, you reference specific files by name in your tasks.

## Variable Priority: defaults/ vs vars/

```
┌─────────────────────────────────────────────────────────┐
│                Variable Precedence                       │
│                                                         │
│  LOW ◄──────────────────────────────────────────► HIGH  │
│                                                         │
│  defaults/main.yml                                      │
│       │                                                 │
│       ▼                                                 │
│  group_vars/all                                         │
│       │                                                 │
│       ▼                                                 │
│  group_vars/group_name                                  │
│       │                                                 │
│       ▼                                                 │
│  host_vars/hostname                                     │
│       │                                                 │
│       ▼                                                 │
│  playbook vars / vars_files                             │
│       │                                                 │
│       ▼                                                 │
│  vars/main.yml            ← Hard to override            │
│       │                                                 │
│       ▼                                                 │
│  extra vars (-e)          ← Always wins                 │
└─────────────────────────────────────────────────────────┘
```

- Use `defaults/` for values users should customize (ports, package names, feature flags).
- Use `vars/` for internal constants that shouldn't change (paths derived from OS, fixed filenames).

## Role Execution Flow

```
ansible-playbook site.yml
         │
         ▼
┌─────────────────────┐
│  Load meta/main.yml │──── Pull in dependency roles first
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ Load defaults/      │──── Set lowest-priority variables
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ Load vars/          │──── Set high-priority variables
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ Execute tasks/      │──── Run tasks in order
│   └── main.yml      │     (can include other .yml files)
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ Flush handlers/     │──── Run notified handlers at end of play
│   └── main.yml      │
└─────────────────────┘
```

## Create a Role

```bash
# Scaffold a new role with the standard directory structure
ansible-galaxy init my_role

# Scaffold in a custom roles path
ansible-galaxy init --init-path ./roles my_role
```

## Example: Minimal nginx Role

```
roles/nginx/
├── tasks/main.yml
├── handlers/main.yml
├── templates/nginx.conf.j2
├── defaults/main.yml
└── meta/main.yml
```

```yaml
# defaults/main.yml
nginx_port: 80
nginx_worker_processes: auto
```

```yaml
# tasks/main.yml
- name: Install nginx
  apt:
    name: nginx
    state: present

- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: restart nginx

- name: Ensure nginx is running
  service:
    name: nginx
    state: started
    enabled: yes
```

```yaml
# handlers/main.yml
- name: restart nginx
  service:
    name: nginx
    state: restarted
```

```yaml
# meta/main.yml
dependencies:
  - role: common
galaxy_info:
  author: your_name
  description: Install and configure nginx
  min_ansible_version: "2.9"
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
```

## Splitting Tasks into Multiple Files

For complex roles, split `tasks/main.yml` into logical files:

```yaml
# tasks/main.yml
- name: Install packages
  import_tasks: install.yml

- name: Configure application
  import_tasks: configure.yml

- name: Set up service
  import_tasks: service.yml
```

```
roles/my_app/
└── tasks/
    ├── main.yml        ← Entry point (imports others)
    ├── install.yml     ← Package installation
    ├── configure.yml   ← Config file deployment
    └── service.yml     ← Service management
```

> `import_tasks` is static (parsed at load time). Use `include_tasks` if you need conditional or looped includes at runtime.
