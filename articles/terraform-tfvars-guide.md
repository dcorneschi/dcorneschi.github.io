# Terraform tfvars: Variable Definitions Reference

Complete reference for defining variable values in `terraform.tfvars` files.

## Basic Types

### String

```hcl
instance_name = "my-instance"
region = "us-east-1"
```

### Number

```hcl
instance_count = 3
port = 8080
disk_size = 100
```

### Bool

```hcl
enable_monitoring = true
is_production = false
```

## Collection Types

### List of Strings

```hcl
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
allowed_ips = ["10.0.0.0/8", "172.16.0.0/12"]
```

### List of Numbers

```hcl
allowed_ports = [80, 443, 8080]
instance_counts = [1, 2, 3, 5]
```

### Set of Strings

```hcl
# Same syntax as list - Terraform ensures uniqueness
security_groups = ["sg-123", "sg-456", "sg-789"]
```

## Map Types

### Map of Strings

```hcl
tags = {
  Environment = "dev"
  Project     = "myapp"
  Owner       = "team-a"
  CostCenter  = "engineering"
}
```

### Map of Numbers

```hcl
instance_sizes = {
  dev     = 2
  staging = 5
  prod    = 10
}
```

### Map of Bools

```hcl
feature_flags = {
  enable_logging    = true
  enable_monitoring = false
  enable_backup     = true
}
```

## Complex Types

### Object

```hcl
instance_config = {
  instance_type = "t3.micro"
  ami_id        = "ami-12345"
  disk_size     = 20
  monitoring    = true
}
```

### Tuple (Ordered, Fixed-Length)

```hcl
# Format: [string, number, bool]
network_config = ["10.0.0.0/16", 100, true]
```

### List of Objects

```hcl
instances = [
  {
    name = "web-server"
    type = "t3.micro"
    size = 20
  },
  {
    name = "db-server"
    type = "t3.small"
    size = 50
  },
  {
    name = "cache-server"
    type = "t3.nano"
    size = 10
  }
]
```

### Map of Objects

```hcl
environments = {
  dev = {
    instance_type  = "t3.micro"
    instance_count = 1
    enable_backup  = false
  }
  staging = {
    instance_type  = "t3.small"
    instance_count = 2
    enable_backup  = true
  }
  prod = {
    instance_type  = "t3.large"
    instance_count = 3
    enable_backup  = true
  }
}
```

### Nested Complex Types

```hcl
# List of objects with nested maps
applications = [
  {
    name = "frontend"
    replicas = 3
    resources = {
      cpu    = "500m"
      memory = "512Mi"
    }
    env_vars = {
      NODE_ENV = "production"
      API_URL  = "https://api.example.com"
    }
  },
  {
    name = "backend"
    replicas = 5
    resources = {
      cpu    = "1000m"
      memory = "1Gi"
    }
    env_vars = {
      DB_HOST = "db.example.com"
      DB_PORT = "5432"
    }
  }
]
```

## Special Cases

### Optional Attributes

```hcl
# Only provide required fields, omit optional ones
server_config = {
  name          = "my-server"
  instance_type = "t3.micro"
  # monitoring and tags are optional, can be omitted
}
```

### Null Values

```hcl
backup_retention = null
snapshot_id = null
```

### Any Type

```hcl
# Can be any valid HCL structure
custom_config = {
  key1 = "value1"
  key2 = 123
  key3 = true
  key4 = ["a", "b"]
}
```

### Sensitive Variables

```hcl
# Same syntax as regular string, but marked sensitive in variable definition
db_password = "super-secret-password"
api_key = "sk-1234567890abcdef"
```

## Quick Reference Table

| Type | Syntax | Example |
|------|--------|---------|
| `string` | `"value"` | `name = "server"` |
| `number` | `123` | `count = 5` |
| `bool` | `true` / `false` | `enabled = true` |
| `list(string)` | `["a", "b"]` | `zones = ["us-east-1a", "us-east-1b"]` |
| `list(number)` | `[1, 2, 3]` | `ports = [80, 443]` |
| `set(string)` | `["a", "b"]` | `groups = ["sg-1", "sg-2"]` |
| `map(string)` | `{ key = "value" }` | `tags = { Env = "dev" }` |
| `map(number)` | `{ key = 123 }` | `sizes = { dev = 2 }` |
| `object({...})` | `{ k1 = v1, k2 = v2 }` | `config = { type = "t3.micro", size = 20 }` |
| `tuple([...])` | `[v1, v2, v3]` | `net = ["10.0.0.0/16", 100, true]` |
| `list(object({...}))` | `[{ k = v }, { k = v }]` | `servers = [{ name = "web", type = "t3.micro" }]` |
| `map(object({...}))` | `{ k1 = {...}, k2 = {...} }` | `envs = { dev = { type = "t3.micro" } }` |

## Key Rules

1. **Strings** — Always use double quotes: `"value"`
2. **Numbers** — No quotes: `123`, `3.14`
3. **Bools** — No quotes: `true`, `false`
4. **Lists/Tuples** — Use square brackets: `[...]`
5. **Maps/Objects** — Use curly braces: `{...}`
6. **Nested structures** — Combine brackets and braces as needed
7. **Trailing commas** — Optional but recommended for multi-line structures
8. **Comments** — Use `#` for single-line or `/* */` for multi-line

## Common Patterns

### Multi-line for readability

```hcl
tags = {
  Environment = "production"
  Project     = "myapp"
  Owner       = "team-a"
  CostCenter  = "engineering"
  Compliance  = "pci-dss"
}
```

### Inline for simple values

```hcl
tags = { Environment = "dev", Project = "test" }
```

### Mixed nesting

```hcl
vpc_config = {
  cidr_block = "10.0.0.0/16"
  subnets = [
    { cidr = "10.0.1.0/24", az = "us-east-1a", public = true },
    { cidr = "10.0.2.0/24", az = "us-east-1b", public = true },
    { cidr = "10.0.3.0/24", az = "us-east-1a", public = false },
    { cidr = "10.0.4.0/24", az = "us-east-1b", public = false }
  ]
  enable_dns = true
  tags = {
    Name = "main-vpc"
    Tier = "network"
  }
}
```

## Best Practices

1. Use meaningful variable names
2. Keep related variables grouped together
3. Add comments to explain complex structures
4. Use consistent formatting and indentation
5. Consider using separate `.tfvars` files per environment
6. Never commit sensitive values to version control
7. Use `.tfvars.example` files as templates
8. Validate syntax with `terraform validate`

## File Naming Conventions

| File | Purpose | Auto-Loaded |
|------|---------|-------------|
| `terraform.tfvars` | Default values | Yes |
| `terraform.tfvars.json` | Default values (JSON format) | Yes |
| `*.auto.tfvars` | Environment-specific (loaded alphabetically) | Yes |
| `*.auto.tfvars.json` | Environment-specific (JSON format) | Yes |
| `dev.tfvars` | Environment-specific (use with `-var-file`) | No |
| `prod.tfvars` | Production values (use with `-var-file`) | No |
| `terraform.tfvars.example` | Template file (commit to git) | No |
| `secrets.tfvars` | Sensitive values (add to `.gitignore`) | No |
