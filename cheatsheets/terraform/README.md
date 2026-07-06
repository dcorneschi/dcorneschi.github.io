<p align="center">
  <img src="/cheatsheets/terraform/images/terraform-logo.svg" alt="Terraform Logo" width="200"/>
</p>

<h1 align="center">Terraform Cheatsheet</h1>

Comprehensive Terraform reference guide featuring installation instructions across multiple platforms, core commands, state management, workspace operations, debugging techniques, and essential utilities for infrastructure as code.

### Installation Commands

| Platform | Method | Command |
|----------|--------|---------|
| **Ubuntu/Debian** | Package Manager | `wget -O - https://apt.releases.hashicorp.com/gpg \| sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg`<br>`echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release \|\| lsb_release -cs) main" \| sudo tee /etc/apt/sources.list.d/hashicorp.list`<br>`sudo apt update && sudo apt install terraform` |
| **CentOS/RHEL** | Package Manager | `sudo yum install -y yum-utils`<br>`sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo`<br>`sudo yum -y install terraform` |
| **Amazon Linux** | Package Manager | `sudo yum install -y yum-utils shadow-utils`<br>`sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo`<br>`sudo yum -y install terraform` |
| **macOS** | Homebrew | `brew tap hashicorp/tap`<br>`brew install hashicorp/tap/terraform` |
| **macOS (Intel)** | Manual | `curl -O https://releases.hashicorp.com/terraform/1.13.0/terraform_1.13.0_darwin_amd64.zip`<br>`unzip terraform_1.13.0_darwin_amd64.zip`<br>`sudo mv terraform /usr/local/bin/` |
| **macOS (Apple Silicon)** | Manual | `curl -O https://releases.hashicorp.com/terraform/1.13.0/terraform_1.13.0_darwin_arm64.zip`<br>`unzip terraform_1.13.0_darwin_arm64.zip`<br>`sudo mv terraform /usr/local/bin/` |
| **All Platforms** | Autocomplete | `terraform -install-autocomplete` |

### Core Commands

| Command | Description |
|---------|-------------|
| `terraform version` | Get the terraform version |
| `terraform init` | Initialize configuration (generate .terraform & .terraform.lock.hcl) |
| `terraform init -backend-config="key=value"` | Initialize with backend configuration |
| `terraform init -backend-config="backend.conf"` | Initialize with backend configuration file |
| `terraform init -backend=false` | Skip backend initialization entirely |
| `terraform init -upgrade` | Initialize and upgrade providers |
| `terraform init -upgrade=hashicorp/aws` | Upgrade only specific providers |
| `terraform init -get-plugins=false` | Initialize without plugins (offline mode) |
| `terraform init -get=false` | Skip getting/updating modules |
| `terraform init -reconfigure` | Reconfigure backend ignoring saved configuration |
| `terraform init -migrate-state` | Migrate state from another backend |
| `terraform init -input=false` | Skip interactive prompts |
| `terraform init -verify-plugins=false` | Skip plugin signature verification |
| `terraform init -lock=false` | Don't hold state lock during migration (dangerous) |
| `terraform init -lock-timeout=120s` | Change state lock timeout |

### Authentication

| Command | Description |
|---------|-------------|
| `terraform login` | Log in to Terraform Cloud |
| `terraform logout` | Log out from Terraform Cloud |
| `terraform login <hostname>` | Log in to a different host |

### Module Management

| Command | Description |
|---------|-------------|
| `terraform get` | Get modules |
| `terraform get -update` | Update modules |
| `terraform graph -type=plan \| grep module` | Show module tree |

### Planning & Validation

| Command | Description |
|---------|-------------|
| `terraform plan` | Create an execution plan |
| `terraform plan -out=tfplan` | Save plan to a file |
| `terraform plan -var-file="prod.tfvars"` | Plan with variable files |
| `terraform plan -var="region=us-west-2"` | Plan with inline variables |
| `terraform plan -replace aws_instance.example` | Replace selected resources |
| `terraform plan -target=module.vpc` | Target specific resources |
| `terraform validate` | Validate configuration files |
| `terraform fmt` | Format configuration files |
| `terraform fmt -check` | Check if input is formatted |
| `terraform fmt -diff` | Display formatting differences |
| `terraform fmt -recursive` | Format files in subdirectories |

### Apply & Deploy

| Command | Description |
|---------|-------------|
| `terraform apply` | Apply changes |
| `terraform apply tfplan` | Apply a saved plan |
| `terraform apply -auto-approve` | Auto-approve without confirmation |
| `terraform apply -target=aws_instance.example` | Apply with specific targets |
| `terraform apply -var="key=value"` | Apply with specific variable |
| `terraform apply -var-file="prod.tfvars"` | Apply with variable files |
| `terraform apply --parallelism=5` | Set number of simultaneous operations |

### Destroy & Cleanup

| Command | Description |
|---------|-------------|
| `terraform destroy` | Destroy all resources |
| `terraform destroy -auto-approve` | Destroy with auto-approve |
| `terraform destroy -target=aws_instance.example` | Destroy specific resources |
| `terraform plan -destroy` | Plan destruction |
| `terraform plan -destroy -out=destroy.tfplan` | Save destroy plan to file |
| `terraform apply destroy.tfplan` | Apply destroy plan |

### State Management

| Command | Description |
|---------|-------------|
| `terraform show` | Show current state |
| `terraform show my.plan` | Inspect the plan |
| `terraform state list` | List resources in state |
| `terraform state list \| wc -l` | Count resources |
| `terraform state list \| cut -d. -f1 \| sort \| uniq -c` | Count resources by type |
| `terraform show -json \| jq '.values.root_module.resources[] \| {address: .address, depends_on: .depends_on}'` | Get all resource dependencies |
| `terraform state show aws_instance.example` | Show specific resource |
| `terraform state rm aws_instance.example` | Remove resource from state |
| `terraform import aws_instance.example i-1234567890abcdef0` | Import existing resource |
| `terraform state mv aws_instance.example aws_instance.new_name` | Move resource in state |
| `terraform state pull` | Pull remote state |
| `terraform state pull > terraform.tfstate` | Save remote state to file |
| `terraform state push terraform.tfstate` | Push local state to remote |
| `terraform force-unlock -force my_lock_id` | Manually unlock state |

### Workspace Management

| Command | Description |
|---------|-------------|
| `terraform workspace list` | List workspaces |
| `terraform workspace new dev` | Create new workspace |
| `terraform workspace select prod` | Select workspace |
| `terraform workspace show` | Show current workspace |
| `terraform workspace delete dev` | Delete workspace |

### Output & Variables

| Command | Description |
|---------|-------------|
| `terraform output` | Show outputs |
| `terraform output instance_ip` | Show specific output |
| `terraform output -json` | Show outputs in JSON format |
| `terraform output -raw <name>` | Show specific output value without quotes |

### Debugging & Troubleshooting

| Command | Description |
|---------|-------------|
| `export TF_LOG=DEBUG; terraform plan` | Enable debug logging |
| `TF_LOG=DEBUG terraform init` | Enable debug logging for specific command |
| `export TF_LOG_PATH=./terraform.log; terraform apply` | Log to file |
| `unset TF_LOG` or `export TF_LOG=` | Disable logging |

**Available Log Levels:** TRACE, DEBUG, INFO, WARN, ERROR (in order of decreasing verbosity)

### Terraform Console Functions

| Function | Command | Description |
|----------|---------|-------------|
| `upper()` | `echo 'upper("hello")' \| terraform console` | Convert string to uppercase |
| `lower()` | `echo 'lower("HELLO")' \| terraform console` | Convert string to lowercase |
| `title()` | `echo 'title("hello world")' \| terraform console` | Convert to title case |
| `trim()` | `echo 'trim("  hello  ", " ")' \| terraform console` | Remove leading/trailing characters |
| `trimspace()` | `echo 'trimspace("  hello  ")' \| terraform console` | Remove leading/trailing whitespace |
| `join()` | `echo 'join(",",[\"a\",\"b\",\"c\"])' \| terraform console` | Join list elements with delimiter |
| `split()` | `echo 'split(",","a,b,c")' \| terraform console` | Split string into list |
| `replace()` | `echo 'replace("hello world", "world", "terraform")' \| terraform console` | Replace substring in string |
| `substr()` | `echo 'substr("hello world", 0, 5)' \| terraform console` | Extract substring |
| `length()` | `echo 'length([\"a\",\"b\",\"c\"])' \| terraform console` | Get length of list/map/string |
| `element()` | `echo 'element([\"a\",\"b\",\"c\"], 1)' \| terraform console` | Get element at index |
| `index()` | `echo 'index([\"a\",\"b\",\"c\"], \"b")' \| terraform console` | Find index of element |
| `contains()` | `echo 'contains([\"a\",\"b\",\"c\"], \"b\")' \| terraform console` | Check if list contains value |
| `concat()` | `echo 'concat([\"a\",\"b\"],[\"c\",\"d\"])' \| terraform console` | Concatenate lists |
| `flatten()` | `echo 'flatten([[\"a\",\"b\"],[\"c\",\"d\"]])' \| terraform console` | Flatten nested lists |
| `distinct()` | `echo 'distinct([\"a\",\"b\",\"a\",\"c\"])' \| terraform console` | Remove duplicate values |
| `sort()` | `echo 'sort([\"c\",\"a\",\"b\"])' \| terraform console` | Sort list alphabetically |
| `reverse()` | `echo 'reverse([\"a\",\"b\",\"c\"])' \| terraform console` | Reverse list order |
| `slice()` | `echo 'slice([\"a\",\"b\",\"c\",\"d\"], 1, 3)' \| terraform console` | Extract slice from list |
| `merge()` | `echo 'merge({a=1},{b=2},{c=3})' \| terraform console` | Merge maps together |
| `keys()` | `echo 'keys({a=1,b=2,c=3})' \| terraform console` | Get map keys as list |
| `values()` | `echo 'values({a=1,b=2,c=3})' \| terraform console` | Get map values as list |
| `lookup()` | `echo 'lookup({a=1,b=2}, "a", "default")' \| terraform console` | Get map value with default |
| `zipmap()` | `echo 'zipmap([\"a\",\"b\"],[1,2])' \| terraform console` | Create map from two lists |
| `base64encode()` | `echo 'base64encode("hello")' \| terraform console` | Encode string to base64 |
| `base64decode()` | `echo 'base64decode("aGVsbG8=")' \| terraform console` | Decode base64 string |
| `jsonencode()` | `echo 'jsonencode({name="test"})' \| terraform console` | Encode value as JSON |
| `jsondecode()` | `echo 'jsondecode("{\"name\":\"test\"}")' \| terraform console` | Decode JSON string |
| `yamlencode()` | `echo 'yamlencode({name="test"})' \| terraform console` | Encode value as YAML |
| `yamldecode()` | `echo 'yamldecode("name: test")' \| terraform console` | Decode YAML string |
| `format()` | `echo 'format("Hello, %s!", "World")' \| terraform console` | Format string with placeholders |
| `formatlist()` | `echo 'formatlist("Hello, %s!", [\"Alice\",\"Bob\"])' \| terraform console` | Format list of strings |
| `regex()` | `echo 'regex("[a-z]+", "abc123")' \| terraform console` | Extract regex match |
| `regexall()` | `echo 'regexall("[0-9]+", "a1b2c3")' \| terraform console` | Extract all regex matches |
| `can()` | `echo 'can(regex("^[a-z]+$", "abc"))' \| terraform console` | Test if expression succeeds |
| `try()` | `echo 'try(1/0, "error")' \| terraform console` | Try expression with fallback |
| `coalesce()` | `echo 'coalesce("", null, "default")' \| terraform console` | Return first non-null/empty value |
| `coalescelist()` | `echo 'coalescelist([], ["default"])' \| terraform console` | Return first non-empty list |
| `compact()` | `echo 'compact([\"a\",\"\",\"b\",null,\"c\"])' \| terraform console` | Remove empty strings from list |
| `chunklist()` | `echo 'chunklist([\"a\",\"b\",\"c\",\"d\"], 2)' \| terraform console` | Split list into chunks |
| `setproduct()` | `echo 'setproduct([\"a\",\"b\"],[1,2])' \| terraform console` | Cartesian product of sets |
| `setunion()` | `echo 'setunion([\"a\",\"b\"],[\"b\",\"c\"])' \| terraform console` | Union of sets |
| `setintersection()` | `echo 'setintersection([\"a\",\"b\"],[\"b\",\"c\"])' \| terraform console` | Intersection of sets |
| `setsubtract()` | `echo 'setsubtract([\"a\",\"b\",\"c\"],[\"b\"])' \| terraform console` | Subtract one set from another |
| `abs()` | `echo 'abs(-5)' \| terraform console` | Absolute value |
| `ceil()` | `echo 'ceil(4.3)' \| terraform console` | Round up to nearest integer |
| `floor()` | `echo 'floor(4.7)' \| terraform console` | Round down to nearest integer |
| `max()` | `echo 'max(5, 12, 9)' \| terraform console` | Maximum value |
| `min()` | `echo 'min(5, 12, 9)' \| terraform console` | Minimum value |
| `pow()` | `echo 'pow(2, 3)' \| terraform console` | Power (2^3) |
| `log()` | `echo 'log(10, 100)' \| terraform console` | Logarithm |
| `parseint()` | `echo 'parseint("100", 10)' \| terraform console` | Parse string to integer |
| `timestamp()` | `echo 'timestamp()' \| terraform console` | Current timestamp |
| `formatdate()` | `echo 'formatdate("YYYY-MM-DD", timestamp())' \| terraform console` | Format timestamp |
| `timeadd()` | `echo 'timeadd(timestamp(), "1h")' \| terraform console` | Add duration to timestamp |
| `timecmp()` | `echo 'timecmp(timestamp(), "2024-01-01T00:00:00Z")' \| terraform console` | Compare timestamps |
| `uuid()` | `echo 'uuid()' \| terraform console` | Generate UUID |
| `uuidv5()` | `echo 'uuidv5("dns", "terraform.io")' \| terraform console` | Generate UUIDv5 |
| `md5()` | `echo 'md5("hello")' \| terraform console` | MD5 hash |
| `sha1()` | `echo 'sha1("hello")' \| terraform console` | SHA1 hash |
| `sha256()` | `echo 'sha256("hello")' \| terraform console` | SHA256 hash |
| `sha512()` | `echo 'sha512("hello")' \| terraform console` | SHA512 hash |
| `bcrypt()` | `echo 'bcrypt("password")' \| terraform console` | Bcrypt hash |
| `fileexists()` | `echo 'fileexists("main.tf")' \| terraform console` | Check if file exists |
| `file()` | `echo 'file("main.tf")' \| terraform console` | Read file content |
| `filebase64()` | `echo 'filebase64("main.tf")' \| terraform console` | Read file as base64 |
| `basename()` | `echo 'basename("/path/to/file.txt")' \| terraform console` | Get filename from path |
| `dirname()` | `echo 'dirname("/path/to/file.txt")' \| terraform console` | Get directory from path |
| `pathexpand()` | `echo 'pathexpand("~/.ssh/id_rsa")' \| terraform console` | Expand ~ in path |
| `cidrhost()` | `echo 'cidrhost("10.0.0.0/24", 5)' \| terraform console` | Calculate IP address in CIDR |
| `cidrnetmask()` | `echo 'cidrnetmask("10.0.0.0/24")' \| terraform console` | Get netmask from CIDR |
| `cidrsubnet()` | `echo 'cidrsubnet("10.0.0.0/16", 8, 2)' \| terraform console` | Calculate subnet address |
| `cidrsubnets()` | `echo 'cidrsubnets("10.0.0.0/16", 4, 4, 8)' \| terraform console` | Calculate multiple subnets |
| `tobool()` | `echo 'tobool("true")' \| terraform console` | Convert to boolean |
| `tolist()` | `echo 'tolist(["a","b"])' \| terraform console` | Convert to list |
| `tomap()` | `echo 'tomap({a=1,b=2})' \| terraform console` | Convert to map |
| `tonumber()` | `echo 'tonumber("42")' \| terraform console` | Convert to number |
| `toset()` | `echo 'toset(["a","b","a"])' \| terraform console` | Convert to set |
| `tostring()` | `echo 'tostring(42)' \| terraform console` | Convert to string |
| `type()` | `echo 'type("hello")' \| terraform console` | Get type of value |

### Graph & Dependencies

| Command | Description |
|---------|-------------|
| `terraform graph` | Generate dependency graph |
| `terraform graph > graph.dot` | Save graph to file |
| `terraform graph \| dot -Tpng > graph.png` | Convert to PNG (requires Graphviz) |
| `terraform graph -draw-cycles` | Show resource dependencies |

### Provider Management

| Command | Description |
|---------|-------------|
| `terraform providers` | Show providers |
| `terraform providers lock` | Lock provider versions |
| `terraform providers lock -platform=linux_amd64 -platform=darwin_amd64 -platform=windows_amd64` | Pre-populate hashes for multiple platforms |
| `terraform providers mirror /path/to/mirror` | Mirror providers for offline use |

### State File Operations

| Command | Description |
|---------|-------------|
| `cp terraform.tfstate terraform.tfstate.backup` | Backup state file |
| `cp terraform.tfstate.backup terraform.tfstate` | Restore state file |
| `terraform show -json > state.json` | Convert state to JSON |
| `diff terraform.tfstate.backup terraform.tfstate` | Compare state files |
| `terraform plan -refresh-only` | Inspect resource drift without updating state |
| `terraform apply -refresh-only` | Plan and refresh the state file |

### Testing & Validation

| Command | Description |
|---------|-------------|
| `terraform validate` | Validate syntax (requires terraform init) |
| `terraform validate -backend=false` | Validate code, skip backend validation |
| `terraform validate -json` | Output in machine-readable JSON format |

### Utilities

#### [tfutils/tfenv](https://github.com/tfutils/tfenv)

```bash
# Install tfenv on macOS
brew install tfenv

# Install tfenv on Linux
git clone --depth=1 https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bash_profile

# List available versions
tfenv list-remote

# Install specific version
tfenv install 1.5.0

# Use specific version
tfenv use 1.5.0

# Install latest version
tfenv install latest

# Show current version
tfenv version
```

#### [terraform-linters/tflint](https://github.com/terraform-linters/tflint)

```bash
# Install tflint on macOS
brew install tflint

# Install tflint on Linux
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Lint with TFLint
tflint
```

#### [aquasecurity/tfsec](https://github.com/aquasecurity/tfsec)

```bash
# Install tfsec on macOS
brew install tfsec

# Install tfsec on Linux
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash

# Scan for security issues (using tfsec)
tfsec .

# Scan specific directory
tfsec ./infrastructure

# Output JSON format
tfsec --format json .
```

#### [terraform-docs/terraform-docs](https://github.com/terraform-docs/terraform-docs)

```bash
# Install terraform-docs on macOS
brew install terraform-docs

# Install terraform-docs on Linux
curl -sSLo ./terraform-docs.tar.gz https://terraform-docs.io/dl/v0.20.0/terraform-docs-v0.20.0-$(uname)-amd64.tar.gz
tar -xzf terraform-docs.tar.gz
chmod +x terraform-docs
mv terraform-docs /some-dir-in-your-PATH/terraform-docs

# Generate documentation (using terraform-docs)
terraform-docs markdown . > README.md
```

### Terraform Variables

```bash
# String variables
export TF_VAR_region="us-west-2"
export TF_VAR_instance_type="t3.micro"
export TF_VAR_environment="production"
export TF_VAR_project_name="my-app"
export TF_VAR_ami_id="ami-0d26eb3972b7f8c96"

# Number variables
export TF_VAR_instance_count="3"
export TF_VAR_max_size="10"
export TF_VAR_port="8080"
export TF_VAR_timeout="300"

# Boolean variables
export TF_VAR_enable_monitoring="true"
export TF_VAR_enable_backup="false"
export TF_VAR_create_dns_record="true"
export TF_VAR_enable_encryption="true"

# List variables (same type elements)
export TF_VAR_availability_zones='["us-west-2a","us-west-2b","us-west-2c"]'
export TF_VAR_allowed_ips='["192.168.1.0/24","10.0.0.0/8"]'
export TF_VAR_subnet_ids='["subnet-abc123","subnet-def456"]'

# Map variables (key-value pairs)
export TF_VAR_tags='{"Environment":"prod","Team":"devops","CostCenter":"engineering"}'
export TF_VAR_instance_types='{"dev":"t3.micro","staging":"t3.small","prod":"t3.large"}'

# Object variables (structured data with different types)
export TF_VAR_server_config='{"name":"web-server","port":8080,"enabled":true}'
export TF_VAR_database='{"engine":"postgres","version":"14.5","multi_az":true,"allocated_storage":100}'

# Tuple variables (fixed-length, mixed types)
export TF_VAR_deployment_info='["v1.2.3",8080,true]'

# Set variables (unique values, unordered)
export TF_VAR_security_groups='["sg-123456","sg-789012","sg-345678"]'

# Complex nested structures
export TF_VAR_vpc_config='{"cidr":"10.0.0.0/16","enable_dns":true,"subnets":[{"cidr":"10.0.1.0/24","az":"us-west-2a"},{"cidr":"10.0.2.0/24","az":"us-west-2b"}]}'

# Map of objects (multiple structured items)
export TF_VAR_subnets='{"subnet_a":{"name":"Public-A","cidr":"10.0.1.0/24","public":true},"subnet_b":{"name":"Private-B","cidr":"10.0.2.0/24","public":false}}'

# List of objects (ordered structured items)
export TF_VAR_instances='[{"name":"web-1","type":"t3.micro","az":"us-west-2a"},{"name":"web-2","type":"t3.micro","az":"us-west-2b"}]'

# Sensitive variables (passwords, keys, tokens)
export TF_VAR_db_password="super-secret-password"
export TF_VAR_api_key="your-api-key-here"
export TF_VAR_private_key="-----BEGIN RSA PRIVATE KEY-----..."

# Terraform-specific environment variables
export TF_LOG=DEBUG                                    # Log level: TRACE, DEBUG, INFO, WARN, ERROR
export TF_LOG_PATH="./terraform.log"                   # Log file path
export TF_LOG_CORE=TRACE                               # Core Terraform logging
export TF_LOG_PROVIDER=DEBUG                           # Provider plugin logging

# Configuration and behavior
export TF_CLI_CONFIG_FILE="$HOME/.terraformrc"         # CLI configuration file
export TF_DATA_DIR=".terraform"                        # Data directory for plugins and modules
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"  # Plugin cache directory
export TF_INPUT=false                                  # Disable interactive input prompts
export TF_IN_AUTOMATION=true                           # Indicate running in CI/CD
export TF_WORKSPACE="production"                       # Set active workspace

# Terraform Cloud/Enterprise
export TF_TOKEN_app_terraform_io="your-token-here"     # Terraform Cloud token
export TF_CLOUD_ORGANIZATION="my-org"                  # Organization name
export TF_CLOUD_HOSTNAME="app.terraform.io"            # Terraform Cloud hostname

# CLI behavior customization
export TF_CLI_ARGS="-no-color"                         # Global CLI arguments
export TF_CLI_ARGS_plan="-refresh=false"               # Arguments for plan command
export TF_CLI_ARGS_apply="-auto-approve"               # Arguments for apply command

# Provider-specific variables (examples)
export TF_VAR_aws_access_key="AKIAIOSFODNN7EXAMPLE"
export TF_VAR_aws_secret_key="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export TF_VAR_azure_subscription_id="00000000-0000-0000-0000-000000000000"
export TF_VAR_gcp_project_id="my-gcp-project"
```

### Variable Precedence Order

Terraform loads variables in the following order (highest to lowest priority - later sources override earlier ones):

| Priority | Source | Example | Notes |
|----------|--------|---------|-------|
| 1 (Highest) | Command-line `-var` flags | `terraform apply -var="region=us-east-1"` | Last `-var` flag wins if multiple are specified |
| 2 | Command-line `-var-file` flags | `terraform apply -var-file="prod.tfvars"` | Last `-var-file` wins if multiple are specified |
| 3 | `*.auto.tfvars` or `*.auto.tfvars.json` files | `production.auto.tfvars` | Loaded automatically in lexical order |
| 4 | `terraform.tfvars.json` file | `terraform.tfvars.json` | Loaded automatically if present |
| 5 | `terraform.tfvars` file | `terraform.tfvars` | Loaded automatically if present |
| 6 | Environment variables | `export TF_VAR_region="us-west-2"` | Must use `TF_VAR_` prefix |
| 7 (Lowest) | Default values in variable blocks | `variable "region" { default = "us-east-1" }` | Used only if no other source provides a value |

**Key Points:**
- CLI flags (`-var` and `-var-file`) always take precedence over everything else
- If a variable is not set by any method, Terraform will prompt interactively (unless `-input=false` is set)
- `*.auto.tfvars` files are processed in alphabetical order
- Multiple `-var` or `-var-file` flags are processed left to right (rightmost wins)
- Environment variables are useful for CI/CD pipelines and sensitive values

### Different Ways to Set Variables

| Method | Command | Use Case |
|--------|---------|----------|
| Inline variable | `terraform plan -var="instance_type=t2.large"` | Quick one-off changes |
| Multiple inline variables | `terraform apply -var="image_id=ami-abc123" -var="instance_type=t2.micro"` | Setting multiple values at once |
| List variable inline | `terraform apply -var='image_id_list=["ami-abc123","ami-def456"]'` | Passing list values |
| Map variable inline | `terraform apply -var='image_id_map={"us-east-1":"ami-abc123","us-east-2":"ami-def456"}'` | Passing map values |
| Variable file | `terraform plan -var-file="prod.tfvars"` | Environment-specific configurations |
| Environment variable | `export TF_VAR_instance_type="t2.small"` | CI/CD pipelines, sensitive values |
| Environment variable with command | `TF_VAR_instance_type="t2.small" terraform plan` | One-time environment variable |
| Default in variable block | `variable "region" { default = "us-east-1" }` | Fallback values |

### Auto-Loading Variable Files

Terraform automatically loads variable files without requiring the `-var-file` flag:

| File Name Pattern | Auto-Loaded | Example |
|-------------------|-------------|---------|
| `terraform.tfvars` | ✅ Yes | `terraform.tfvars` |
| `terraform.tfvars.json` | ✅ Yes | `terraform.tfvars.json` |
| `*.auto.tfvars` | ✅ Yes | `dev.auto.tfvars`, `prod.auto.tfvars` |
| `*.auto.tfvars.json` | ✅ Yes | `dev.auto.tfvars.json` |
| Custom name (e.g., `prod.tfvars`) | ❌ No | Requires `-var-file="prod.tfvars"` |

**Best Practices:**
- Use `terraform.tfvars` for default/common values
- Use `*.auto.tfvars` for environment-specific values (e.g., `dev.auto.tfvars`, `prod.auto.tfvars`)
- Use custom-named `.tfvars` files with `-var-file` flag for explicit control
- Never commit sensitive values in `.tfvars` files to version control
- Use environment variables (`TF_VAR_*`) for secrets and CI/CD pipelines

### State Management Best Practices

```bash
# Backup state before major operations
cp terraform.tfstate terraform.tfstate.backup.$(date +%Y%m%d_%H%M%S)

# Use remote state
terraform init -backend-config="bucket=my-tf-state" -backend-config="key=prod/terraform.tfstate"

# Enable state locking
terraform init -backend-config="dynamodb_table=terraform-locks"
```

### Useful Aliases

Add these to your `~/.zshrc` or `~/.bashrc`:

```bash
alias tf="terraform"
alias tfi="terraform init"
alias tfp="terraform plan"
alias tfa="terraform apply"
alias tfd="terraform destroy"
alias tfs="terraform show"
alias tfo="terraform output"
alias tfv="terraform validate"
alias tff="terraform fmt"
alias tfw="terraform workspace"
alias tfws="terraform workspace show"
alias tfsl="terraform state list"
```

### Useful Functions

```bash
# Generic function for any base64 string
tf_base64_decode() {
    if [ $# -eq 0 ]; then
        echo "Usage: tf_base64_decode '<base64_string>'"
        return 1
    fi
    echo "base64decode(\"$1\")" | terraform console
}
```

### Best Practices & tips

1. Always commit lock files to version control alongside your Terraform configuration files.
2. Never add `.terraform.lock.hcl to` `.gitignore` - ignoring lock files defeats their purpose.
3. Never commit .tfvars files with secrets.
3. Store state files remotely and securely.
5. Use workspaces for environment separation.
6. Use modules for reusable components.
7. Pin provider versions in production.
8. Always run `terraform plan` before `apply`.
9. Review and understand the execution plan before applying.
