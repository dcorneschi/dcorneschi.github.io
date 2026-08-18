# SSH Heredoc Variable Expansion

Using heredocs with SSH to pass variables, create files, and run multi-line scripts on remote servers. Understanding when variables expand locally vs remotely is essential for CI/CD pipelines and deployment scripts.

## Key Rules

| Syntax | Behavior |
|--------|----------|
| `<< EOF` | Variables expand **locally** before sending |
| `<< 'EOF'` | Variables expand **on the remote** side |
| `\$VAR` | Escaped — prevents local expansion in unquoted heredoc |

## Basic EOF with Variable Substitution

Variables are expanded locally before the commands are sent to the remote server:

```bash
ssh user@server << EOF
export MY_VAR="$LOCAL_VAR"
export COMMIT_SHA="$CI_COMMIT_SHA"
echo "Variables set: MY_VAR=$MY_VAR"
EOF
```

## Quoted EOF (Prevents Local Expansion)

Quoting the delimiter sends everything literally — variables expand on the remote side:

```bash
ssh user@server << 'EOF'
export MY_VAR="value_set_on_remote"
export CURRENT_TIME=$(date)
echo "Set on remote: $MY_VAR at $CURRENT_TIME"
EOF
```

## Mixed Variable Expansion (Local + Remote)

Combine local and remote expansion by escaping remote variables with `\$`:

```bash
ssh user@server << EOF
# These expand locally before sending
export LOCAL_COMMIT="$CI_COMMIT_SHA"
export LOCAL_PIPELINE="$CI_PIPELINE_ID"

# These expand on remote
export REMOTE_TIME=\$(date)
export REMOTE_USER=\$(whoami)
export REMOTE_HOST=\$(hostname)

echo "Local vars: $LOCAL_COMMIT, $LOCAL_PIPELINE"
echo "Remote vars: \$REMOTE_TIME, \$REMOTE_USER, \$REMOTE_HOST"
EOF
```

## Creating Files with EOF

```bash
ssh user@server << EOF
cat > /path/to/config.env << 'ENVEOF'
DATABASE_URL=$DATABASE_URL
API_KEY=$API_KEY
COMMIT_SHA=$CI_COMMIT_SHA
DEPLOY_TIME=\$(date -Iseconds)
ENVEOF

chmod 600 /path/to/config.env
EOF
```

## Nested Heredocs

Use different delimiters for inner heredocs when creating multiple files:

```bash
ssh user@server << EOF
cat > /app/.env << 'INNER_EOF'
# Application Configuration
NODE_ENV=production
DATABASE_URL=$DATABASE_URL
API_ENDPOINT=$API_ENDPOINT
COMMIT_SHA=$CI_COMMIT_SHA
PIPELINE_ID=$CI_PIPELINE_ID
DEPLOY_TIMESTAMP=$(date +%s)
INNER_EOF

cat > /app/deploy-info.json << 'JSON_EOF'
{
  "commit": "$CI_COMMIT_SHA",
  "pipeline": "$CI_PIPELINE_ID",
  "branch": "$CI_COMMIT_BRANCH",
  "deployer": "$GITLAB_USER_EMAIL",
  "timestamp": "$(date -Iseconds)"
}
JSON_EOF
EOF
```

## Using envsubst with EOF

Prepare a template locally, then substitute and send:

```bash
# First, prepare template
cat << 'TEMPLATE_EOF' > /tmp/config_template
DATABASE_URL=${DATABASE_URL}
API_KEY=${API_KEY}
COMMIT_SHA=${CI_COMMIT_SHA}
ENVIRONMENT=${ENVIRONMENT:-production}
TEMPLATE_EOF

# Then send and process on remote
ssh user@server << EOF
cat > /app/.env << 'CONFIG_EOF'
$(envsubst < /tmp/config_template)
CONFIG_EOF
rm -f /tmp/config_template
EOF
```

## Conditional Variable Setting

```bash
ssh user@server << EOF
# Set variables based on conditions
if [ "$CI_COMMIT_BRANCH" = "main" ]; then
    export ENVIRONMENT="production"
    export DEBUG_MODE="false"
else
    export ENVIRONMENT="staging"
    export DEBUG_MODE="true"
fi

export COMMIT_SHA="$CI_COMMIT_SHA"
export BRANCH="$CI_COMMIT_BRANCH"

# Save to file
cat > /app/runtime.env << 'RUNTIME_EOF'
ENVIRONMENT=\$ENVIRONMENT
DEBUG_MODE=\$DEBUG_MODE
COMMIT_SHA=\$COMMIT_SHA
BRANCH=\$BRANCH
DEPLOY_TIME=\$(date)
RUNTIME_EOF
EOF
```

## Multi-line Variable Values

```bash
CERTIFICATE="-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAKoK/heBjcOuMA0GCSqGSIb3DQEBBQUAMEUxCzAJBgNV
...certificate content...
-----END CERTIFICATE-----"

ssh user@server << EOF
cat > /etc/ssl/certs/app.crt << 'CERT_EOF'
$CERTIFICATE
CERT_EOF

chmod 600 /etc/ssl/certs/app.crt
EOF
```

## Array and List Variables

```bash
ALLOWED_HOSTS="host1.com,host2.com,host3.com"
DATABASE_REPLICAS="db1.example.com db2.example.com db3.example.com"

ssh user@server << EOF
# Convert comma-separated to array
IFS=',' read -ra HOSTS <<< "$ALLOWED_HOSTS"
for host in "\${HOSTS[@]}"; do
    echo "Allowed host: \$host"
done

# Space-separated array
REPLICAS=($DATABASE_REPLICAS)
echo "Primary replica: \${REPLICAS[0]}"

# Save configuration
cat > /app/hosts.conf << 'HOSTS_EOF'
$(echo "$ALLOWED_HOSTS" | tr ',' '\n')
HOSTS_EOF
EOF
```

## Escaping Special Characters

```bash
PASSWORD='p@$$w0rd!with$pecial&chars'
DATABASE_URL='postgresql://user:p@$$w0rd@localhost:5432/db'

ssh user@server << EOF
# Properly escape special characters
export PASSWORD='$PASSWORD'
export DATABASE_URL='$DATABASE_URL'

# Alternative: use printf for complex strings
printf 'DATABASE_URL=%q\n' '$DATABASE_URL' > /app/.env
printf 'PASSWORD=%q\n' '$PASSWORD' >> /app/.env
EOF
```

## Variables with Defaults

```bash
ssh user@server << EOF
cat > /app/.env << 'ENV_EOF'
# Variables with defaults
DATABASE_URL=\${DATABASE_URL:-postgresql://localhost:5432/myapp}
REDIS_URL=\${REDIS_URL:-redis://localhost:6379}
LOG_LEVEL=\${LOG_LEVEL:-info}
WORKER_COUNT=\${WORKER_COUNT:-4}

# Variables from CI/CD
COMMIT_SHA=$CI_COMMIT_SHA
PIPELINE_ID=$CI_PIPELINE_ID
BRANCH=$CI_COMMIT_BRANCH
DEPLOY_TIME=\$(date -Iseconds)
ENV_EOF
EOF
```

## JSON Configuration with Variables

```bash
ssh user@server << EOF
cat > /app/config.json << 'JSON_EOF'
{
  "database": {
    "url": "$DATABASE_URL",
    "pool_size": 10
  },
  "api": {
    "key": "$API_KEY",
    "endpoint": "$API_ENDPOINT"
  },
  "deployment": {
    "commit": "$CI_COMMIT_SHA",
    "pipeline": "$CI_PIPELINE_ID",
    "branch": "$CI_COMMIT_BRANCH",
    "timestamp": "$(date -Iseconds)",
    "deployer": "$GITLAB_USER_EMAIL"
  },
  "runtime": {
    "environment": "${ENVIRONMENT:-production}",
    "debug": ${DEBUG_MODE:-false}
  }
}
JSON_EOF
EOF
```

## YAML Configuration with Variables

```bash
ssh user@server << EOF
cat > /app/config.yaml << 'YAML_EOF'
database:
  url: "$DATABASE_URL"
  pool_size: 10

api:
  key: "$API_KEY"
  endpoint: "$API_ENDPOINT"

deployment:
  commit: "$CI_COMMIT_SHA"
  pipeline: "$CI_PIPELINE_ID"
  branch: "$CI_COMMIT_BRANCH"
  timestamp: "$(date -Iseconds)"
  deployer: "$GITLAB_USER_EMAIL"

runtime:
  environment: "${ENVIRONMENT:-production}"
  debug: ${DEBUG_MODE:-false}
YAML_EOF
EOF
```

## GitLab CI/CD Example

```yaml
deploy:
  stage: deploy
  before_script:
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - ssh-keyscan $SERVER_HOST >> ~/.ssh/known_hosts
  script:
    - |
      ssh user@$SERVER_HOST << EOF
      cat > /app/.env << 'APPENV_EOF'
      NODE_ENV=production
      DATABASE_URL=$DATABASE_URL
      API_KEY=$API_KEY
      COMMIT_SHA=$CI_COMMIT_SHA
      PIPELINE_ID=$CI_PIPELINE_ID
      BRANCH=$CI_COMMIT_BRANCH
      DEPLOY_TIME=\$(date -Iseconds)
      APPENV_EOF

      chmod 600 /app/.env
      chown app:app /app/.env

      # Restart application
      systemctl restart myapp
      EOF
```

## Quick Reference

| Pattern | Expands Where | Use Case |
|---------|---------------|----------|
| `$VAR` in `<< EOF` | Locally | Pass CI/CD variables to remote |
| `$VAR` in `<< 'EOF'` | Remotely | Use remote environment values |
| `\$VAR` in `<< EOF` | Remotely | Mix local and remote expansion |
| `\$(cmd)` in `<< EOF` | Remotely | Run commands on remote |
| `$(cmd)` in `<< EOF` | Locally | Embed local command output |

## Common Use Cases

- **Deployment configuration** — save CI/CD variables to application config files
- **Environment setup** — configure remote servers with local values
- **Multi-environment deploys** — use conditionals to set environment-specific values
- **Configuration management** — generate config files from templates
- **Backup metadata** — save deployment information for rollbacks

## Best Practices

1. Use quoted EOF (`'EOF'`) when all variables should expand remotely
2. Use unquoted EOF when passing local/CI variables to the remote
3. Always set `chmod 600` on files containing secrets
4. Use different delimiter names for nested heredocs
5. Test with `cat` instead of `ssh` during development to see what gets sent
6. Never expose sensitive data in logs or process lists
7. Prefer `printf %q` for values with special characters
