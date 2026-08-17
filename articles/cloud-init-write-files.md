# cloud-init: Different Ways to Create Files on a Server

All the methods available in cloud-init to write, generate, or deploy files locally on a server during provisioning — from simple inline content to fetched remote files and template rendering.

## write_files Module (Recommended)

The `write_files` module is the primary and most flexible way to create files with cloud-init. It runs during the **init** stage (network available) by default.

### Basic Inline Content

```yaml
#cloud-config
write_files:
  - path: /etc/myapp/config.yaml
    content: |
      database:
        host: localhost
        port: 5432
      logging:
        level: info
    owner: root:root
    permissions: '0644'
```

### Binary Content with Encoding

For binary or special content, use encoding to embed it safely in YAML.

```yaml
#cloud-config
write_files:
  - path: /usr/local/bin/myscript.sh
    encoding: b64
    content: IyEvYmluL2Jhc2gKZWNobyAiSGVsbG8gZnJvbSBjbG91ZC1pbml0Ig==
    owner: root:root
    permissions: '0755'
```

Supported encodings:

| Encoding | Description |
|----------|-------------|
| `b64` / `base64` | Base64 encoded content |
| `gz` / `gzip` | Gzip compressed content |
| `gz+b64` / `gz+base64` | Gzip compressed then base64 encoded |
| `text/plain` | Plain text (default) |

### Gzip + Base64 for Large Files

Useful when you need to embed large config files without bloating user-data.

```yaml
#cloud-config
write_files:
  - path: /etc/large-config.json
    encoding: gz+base64
    content: H4sIAAAAAAAAA6tWKkktLlGyUlAqS8wpTtVRKs9ILUpVslJQcgwO8fRzVaoFABnkMKQlAAAA
    owner: root:root
    permissions: '0644'
```

To generate gz+base64 content from your local machine:

```bash
cat large-config.json | gzip | base64
```

### Append to Existing Files

Instead of overwriting, append content to an existing file.

```yaml
#cloud-config
write_files:
  - path: /etc/hosts
    append: true
    content: |
      10.0.1.10 app-server-1
      10.0.1.11 app-server-2
```

### Deferred Write (Final Stage)

By default, `write_files` runs at the init stage. Use `defer: true` to delay writing until the final stage — useful when the target directory is created by a package installed later.

```yaml
#cloud-config
write_files:
  - path: /etc/myapp/settings.conf
    defer: true
    content: |
      [server]
      listen = 0.0.0.0:8080
    owner: myapp:myapp
    permissions: '0640'

packages:
  - myapp
```

### Source (Fetch from URL)

Starting with cloud-init 24.3, you can fetch file content from a remote URL instead of embedding it inline.

```yaml
#cloud-config
write_files:
  - path: /etc/ssl/certs/my-ca.crt
    source:
      uri: https://internal-ca.example.com/ca.crt
      headers:
        Authorization: Bearer my-token
    owner: root:root
    permissions: '0644'
```

## runcmd with Shell Redirects

Use `runcmd` when you need dynamic content, command output, or logic that `write_files` can't express.

### Simple File Creation

```yaml
#cloud-config
runcmd:
  - echo "HOSTNAME=$(hostname)" > /etc/myapp/identity.conf
  - cat <<'EOF' > /etc/systemd/system/myapp.service
    [Unit]
    Description=My Application
    After=network.target

    [Service]
    ExecStart=/usr/local/bin/myapp
    Restart=always

    [Install]
    WantedBy=multi-user.target
    EOF
  - systemctl daemon-reload
```

### Using tee for Multi-Line Content

```yaml
#cloud-config
runcmd:
  - |
    tee /etc/sysctl.d/99-custom.conf > /dev/null <<'EOF'
    net.ipv4.ip_forward = 1
    net.core.somaxconn = 65535
    vm.swappiness = 10
    EOF
  - sysctl --system
```

### Dynamic Content from Commands

```yaml
#cloud-config
runcmd:
  - 'echo "instance_id: $(cloud-init query instance-id)" > /etc/myapp/metadata.yaml'
  - 'echo "local_ip: $(hostname -I | awk "{print \$1}")" >> /etc/myapp/metadata.yaml'
  - 'echo "region: $(cloud-init query region)" >> /etc/myapp/metadata.yaml'
```

## bootcmd (Early Boot, No Network)

Use `bootcmd` when you need files created before networking is configured. Runs on **every boot** unless guarded.

```yaml
#cloud-config
bootcmd:
  - |
    if [ ! -f /etc/cloud/cloud-init.custom-early ]; then
      echo "nameserver 10.0.0.2" > /etc/resolv.conf.early
      touch /etc/cloud/cloud-init.custom-early
    fi
```

> **Warning:** `bootcmd` runs on every boot by default. Guard with a sentinel file or use `cloud_init_modules` frequency settings.

## Scripts (Part Handler / User-Data Script)

### Shell Script as User-Data

If your entire user-data is a script (starts with `#!`), it will be executed and can create files directly.

```bash
#!/bin/bash
mkdir -p /opt/myapp/config

cat > /opt/myapp/config/app.env <<'EOF'
APP_PORT=8080
APP_ENV=production
DB_HOST=10.0.1.5
LOG_LEVEL=warn
EOF

chown -R appuser:appuser /opt/myapp
chmod 600 /opt/myapp/config/app.env
```

### Multi-Part MIME (Combine Scripts + Cloud-Config)

Combine `write_files` with scripts in a single user-data payload using multi-part MIME.

```bash
# Generate with cloud-init's built-in tool
cloud-init devel make-mime \
  --attach config.yaml:cloud-config \
  --attach setup.sh:x-shellscript \
  > user-data.txt
```

Example parts:

**config.yaml:**
```yaml
#cloud-config
write_files:
  - path: /etc/myapp/static.conf
    content: |
      static_key = static_value
```

**setup.sh:**
```bash
#!/bin/bash
# Dynamic file that depends on instance metadata
INSTANCE_ID=$(cloud-init query instance-id)
echo "node_name = ${INSTANCE_ID}" > /etc/myapp/dynamic.conf
```

## Jinja Templates

cloud-init supports Jinja2 templating in cloud-config, allowing instance metadata to be interpolated into file content.

### Template Header

Add `## template: jinja` as the first line of your cloud-config.

```yaml
## template: jinja
#cloud-config
write_files:
  - path: /etc/myapp/instance.conf
    content: |
      hostname = {{ ds.meta_data.local_hostname }}
      instance_id = {{ v1.instance_id }}
      region = {{ v1.region }}
      cloud = {{ v1.cloud_name }}
    permissions: '0644'
```

### Available Jinja Variables

| Variable | Description |
|----------|-------------|
| `v1.instance_id` | Instance ID |
| `v1.region` | Cloud region |
| `v1.cloud_name` | Cloud provider name |
| `v1.local_hostname` | Local hostname |
| `ds.meta_data.*` | Full datasource metadata |

## write_files Field Reference

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `path` | Yes | — | Absolute path for the file |
| `content` | No | `""` | File content (string) |
| `owner` | No | `root:root` | Owner and group (`user:group`) |
| `permissions` | No | `0644` | File permissions (octal string) |
| `encoding` | No | `text/plain` | Content encoding |
| `append` | No | `false` | Append instead of overwrite |
| `defer` | No | `false` | Write at final stage instead of init |
| `source` | No | — | Fetch content from a URL (cloud-init 24.3+) |

## Comparison of Methods

| Method | Stage | Dynamic Content | Runs Every Boot | Best For |
|--------|-------|-----------------|-----------------|----------|
| `write_files` | init / final | No (static) | No (once) | Config files, certs, scripts |
| `write_files` + jinja | init / final | Instance metadata | No (once) | Templates with instance info |
| `runcmd` | final | Yes | No (once) | Files depending on command output |
| `bootcmd` | local (early) | Limited | Yes | Files needed before networking |
| Shell script | final | Yes | No (once) | Complex logic, multi-step setup |

## Why runcmd Is Better for User Home Directory Files

A common mistake is using `write_files` to drop something like `.bash_profile` or `.bashrc` into a user's home directory. The problem: `write_files` runs during the **init** stage, but the `users` module — which actually creates the user and their home directory — runs in the **same stage**. Module ordering within a stage is not guaranteed across all distros and cloud-init versions, so the home directory may not exist yet when `write_files` tries to write.

Even when it works by luck, `write_files` runs as root. The file ends up owned by `root:root` unless you explicitly set `owner`. And if you set `owner` to a user that doesn't exist yet, it fails silently or falls back to root.

### The Problem

```yaml
#cloud-config
users:
  - name: deploy
    shell: /bin/bash
    groups: sudo

# This may FAIL — /home/deploy might not exist yet
write_files:
  - path: /home/deploy/.bash_profile
    owner: deploy:deploy
    permissions: '0644'
    content: |
      export PATH="$HOME/.local/bin:$PATH"
      export EDITOR=vim
      alias ll='ls -la'
```

### The Fix: Use runcmd

`runcmd` runs at the **final** stage — after users are created, packages are installed, and the filesystem is fully set up. This guarantees the home directory exists and the user is available for `chown`.

```yaml
#cloud-config
users:
  - name: deploy
    shell: /bin/bash
    groups: sudo

runcmd:
  - |
    cat > /home/deploy/.bash_profile <<'EOF'
    export PATH="$HOME/.local/bin:$PATH"
    export EDITOR=vim
    alias ll='ls -la'
    EOF
  - chown deploy:deploy /home/deploy/.bash_profile
  - chmod 0644 /home/deploy/.bash_profile
```

### Alternative: write_files with defer

If you prefer the cleaner `write_files` syntax, use `defer: true` to delay the write until the final stage. The user and home directory will exist by then.

```yaml
#cloud-config
users:
  - name: deploy
    shell: /bin/bash
    groups: sudo

write_files:
  - path: /home/deploy/.bash_profile
    defer: true
    owner: deploy:deploy
    permissions: '0644'
    content: |
      export PATH="$HOME/.local/bin:$PATH"
      export EDITOR=vim
      alias ll='ls -la'
```

> **Rule of thumb:** Any file that targets a non-root user's home directory should use either `runcmd` or `write_files` with `defer: true`.

## Tips and Gotchas

- **Quoting permissions** — Always quote permissions in YAML: `'0644'` not `0644`. YAML interprets unquoted octal as decimal.
- **Parent directories** — `write_files` creates parent directories automatically. No need for `mkdir -p` first.
- **Order of execution** — `write_files` (init stage) runs before `runcmd` (final stage). If a runcmd needs a file, `write_files` is a safe choice.
- **Deferred writes** — Use `defer: true` when the target path depends on a package that hasn't been installed yet.
- **Large user-data** — Most cloud providers limit user-data to 16 KB. Use `gz+base64` encoding or fetch from a URL/S3 for large files.
- **Sensitive content** — Avoid putting secrets directly in user-data (it's often visible in instance metadata). Use a secrets manager and fetch at runtime, or restrict metadata access.
- **File overwrite** — `write_files` overwrites existing files by default. Use `append: true` if you want to add to an existing file.
