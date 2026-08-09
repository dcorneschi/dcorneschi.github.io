# JSON Query Tools: JMESPath vs jq vs JSONPath

Three common tools for querying and transforming JSON data. Each has different strengths and use cases.

## Quick Comparison

| | JMESPath | jq | JSONPath |
|--|----------|-----|----------|
| Used by | AWS CLI (`--query`) | CLI tool (standalone) | Kubernetes (`kubectl`), REST APIs |
| Type | Query language | Query + transformation language | Query language |
| Install | Built into AWS CLI | `apt install jq` / `brew install jq` | Built into kubectl, various libraries |
| Strengths | Filtering, formatting AWS output | Full JSON transformation, scripting | Simple path-based access |
| Limitations | Read-only, no creation of new structures | Requires piping, separate tool | Limited filtering, no functions |
| Root symbol | (implicit) | `.` | `$` |
| Wildcard | `[*]` or `[]` | `.[].field` | `[*]` or `..` |
| Filter | `[?expr]` | `select(expr)` | `[?(@.expr)]` |
| Literals | Backticks `` `value` `` | Direct values | N/A |

## JMESPath

Used by AWS CLI natively. Embedded in `--query` parameter.

### Syntax

```bash
aws ec2 describe-instances --query 'Reservations[].Instances[].InstanceId' --output text
```

### Key Features

```bash
# Select fields
--query 'Items[].{Name:name, ID:id}'

# Filter
--query 'Items[?status==`active`]'

# Sort
--query 'sort_by(Items, &date)'

# Functions
--query 'length(Items)'
--query 'max_by(Items, &size)'

# Pipe
--query 'Items[] | [0]'

# Flatten
--query 'Outer[].Inner[].field'
```

### Strengths

- Zero setup for AWS workflows
- Clean syntax for filtering and selecting
- Multi-select hash gives nice table headers
- Built-in functions (sort_by, max_by, contains, length, join)

### Limitations

- Read-only — cannot create new keys, merge objects, or do arithmetic
- No conditional logic (if/then)
- No regex support
- Limited string manipulation

See [JMESPath Query Guide](articles/aws-jmespath-guide.md) for a full reference.

## jq

A standalone command-line JSON processor. The most powerful option for transforming JSON.

### Install

```bash
# Debian/Ubuntu
sudo apt install jq

# RHEL/CentOS
sudo yum install jq

# macOS
brew install jq

# Verify
jq --version
```

### Basic Syntax

```bash
# Identity (pretty-print)
echo '{"name":"test"}' | jq '.'

# Select a field
echo '{"name":"test","id":1}' | jq '.name'
# "test"

# Raw output (no quotes)
echo '{"name":"test"}' | jq -r '.name'
# test

# Select from array
echo '[{"name":"a"},{"name":"b"}]' | jq '.[0].name'
# "a"

# Iterate array
echo '[{"name":"a"},{"name":"b"}]' | jq '.[].name'
# "a"
# "b"
```

### Filtering

```bash
# select() — filter elements
echo '[{"name":"a","active":true},{"name":"b","active":false}]' | \
    jq '.[] | select(.active == true)'

# Multiple conditions
cat data.json | jq '.items[] | select(.status == "running" and .type == "t3.micro")'

# Negation
cat data.json | jq '.items[] | select(.status != "terminated")'

# String matching
cat data.json | jq '.items[] | select(.name | test("^web-"))'

# Contains
cat data.json | jq '.items[] | select(.name | contains("prod"))'
```

### Selecting and Reshaping

```bash
# Pick specific fields
cat data.json | jq '.items[] | {name: .name, id: .id}'

# Rename fields
cat data.json | jq '.items[] | {server_name: .Name, server_id: .InstanceId}'

# Array of arrays (like JMESPath multi-select list)
cat data.json | jq '.items[] | [.name, .id, .status]'

# Collect into array
cat data.json | jq '[.items[] | .name]'
```

### Transformation

```bash
# String interpolation
cat data.json | jq -r '.items[] | "\(.name) — \(.status)"'

# Arithmetic
echo '{"price":10,"qty":3}' | jq '.price * .qty'
# 30

# Conditional
cat data.json | jq '.items[] | if .active then .name else empty end'

# Add/modify fields
echo '{"name":"test"}' | jq '. + {env: "prod", version: 2}'

# Delete fields
echo '{"name":"test","tmp":"x"}' | jq 'del(.tmp)'

# Map (transform each element)
echo '[1,2,3]' | jq 'map(. * 2)'
# [2,4,6]

# Reduce
echo '[1,2,3,4,5]' | jq 'reduce .[] as $x (0; . + $x)'
# 15
```

### Sorting and Grouping

```bash
# Sort by field
cat data.json | jq '.items | sort_by(.name)'

# Sort descending
cat data.json | jq '.items | sort_by(.date) | reverse'

# Group by field
cat data.json | jq '.items | group_by(.status)'

# Unique values
cat data.json | jq '[.items[].status] | unique'

# Count per group
cat data.json | jq '.items | group_by(.status) | map({status: .[0].status, count: length})'
```

### String Functions

```bash
# Split
echo '"a,b,c"' | jq 'split(",")'
# ["a","b","c"]

# Join
echo '["a","b","c"]' | jq 'join("-")'
# "a-b-c"

# Replace (regex)
echo '"hello world"' | jq 'gsub("world"; "jq")'
# "hello jq"

# Replace first occurrence only
echo '"hello world world"' | jq 'sub("world"; "jq")'
# "hello jq world"

# Uppercase/lowercase
echo '"hello"' | jq 'ascii_upcase'
# "HELLO"
echo '"HELLO"' | jq 'ascii_downcase'
# "hello"

# String length
echo '"hello"' | jq 'length'
# 5

# Trim prefix/suffix
echo '"  hello  "' | jq 'ltrimstr(" ") | rtrimstr(" ")'

# Regex test (returns true/false)
echo '"web-server-01"' | jq 'test("^web-")'
# true

# Regex match with captures
echo '"2024-01-15"' | jq 'capture("(?<year>\\d{4})-(?<month>\\d{2})-(?<day>\\d{2})")'
# {"year":"2024","month":"01","day":"15"}

# Case-insensitive regex search with shell variable
jq --arg q "$QUERY" '.[] | select(.name | test($q; "i"))' data.json
```

### Encoding and Formatting

```bash
# Base64 encode
echo '"hello world"' | jq '@base64'
# "aGVsbG8gd29ybGQ="

# Base64 decode
echo '"aGVsbG8gd29ybGQ="' | jq '@base64d'
# "hello world"

# Kubernetes secret decode
kubectl get secret mysecret -o json | jq -r '.data.password | @base64d'

# URL encode
echo '"hello world & more"' | jq '@uri'
# "hello%20world%20%26%20more"

# HTML escape
echo '"<script>alert(1)</script>"' | jq '@html'
# "&lt;script&gt;alert(1)&lt;/script&gt;"

# Serialize as JSON string (for embedding)
echo '{"a":1}' | jq '@json'
# "{\"a\":1}"

# Convert to string
echo '42' | jq 'tostring'
# "42"

# Convert to number
echo '"42"' | jq 'tonumber'
# 42
```

### jq with AWS CLI

```bash
# Pipe AWS JSON output to jq
aws ec2 describe-instances --output json | \
    jq '.Reservations[].Instances[] | {id: .InstanceId, type: .InstanceType, state: .State.Name}'

# Get running instance IDs
aws ec2 describe-instances --output json | \
    jq -r '.Reservations[].Instances[] | select(.State.Name == "running") | .InstanceId'

# Get Name tag
aws ec2 describe-instances --output json | \
    jq -r '.Reservations[].Instances[] | 
        {id: .InstanceId, name: (.Tags // [] | map(select(.Key == "Name")) | .[0].Value // "N/A")}'

# Count instances by state
aws ec2 describe-instances --output json | \
    jq '.Reservations[].Instances[] | .State.Name' | sort | uniq -c

# Transform to CSV
aws ec2 describe-instances --output json | \
    jq -r '.Reservations[].Instances[] | [.InstanceId, .InstanceType, .State.Name] | @csv'

# Transform to TSV
aws ec2 describe-instances --output json | \
    jq -r '.Reservations[].Instances[] | [.InstanceId, .InstanceType] | @tsv'
```

### jq with Docker

```bash
# Get container IPs
docker inspect myapp | jq '.[0].NetworkSettings.Networks | to_entries[] | {network: .key, ip: .value.IPAddress}'

# Get mount points
docker inspect myapp | jq '.[0].Mounts[] | {type: .Type, src: .Source, dst: .Destination}'

# Get environment variables
docker inspect myapp | jq '.[0].Config.Env[]'

# Parse docker compose config
docker compose config --format json | jq '.services | keys[]'
```

### jq with Kubernetes (kubectl)

```bash
# Get pod names and status
kubectl get pods -o json | jq -r '.items[] | "\(.metadata.name) \(.status.phase)"'

# Get containers and their images
kubectl get pods -o json | jq '.items[].spec.containers[] | {name: .name, image: .image}'

# Get nodes with capacity
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, cpu: .status.capacity.cpu, memory: .status.capacity.memory}'
```

### jq Path Exploration (Kubernetes)

Discover the structure of any JSON by exploring its paths:

```bash
# List ALL paths in compact format
kubectl get pods -A -o json | jq -c paths

# Paths as dot notation strings (leaf values only)
kubectl get pods -A -o json | jq -r 'paths(scalars) as $p | $p | join(".")'

# Get all leaf values with their paths
kubectl get pods -A -o json | jq -r 'paths(scalars) as $p | "\($p | join(".")): \(getpath($p))"'
```

### Filtering Paths

```bash
# Only paths containing "status"
kubectl get pods -A -o json | jq -c 'paths | select(. | join(".") | contains("status"))'

# Paths ending with a specific key
kubectl get pods -A -o json | jq -c 'paths | select(.[-1] == "name")'

# Paths at a specific depth
kubectl get pods -A -o json | jq -c 'paths | select(length == 3)'

# Find where restart counts are stored
kubectl get pods -A -o json | jq -c 'paths | select(. | join(".") | contains("restart"))'

# Find all timestamp fields
kubectl get pods -A -o json | jq -c 'paths | select(. | join(".") | test("time|Time"))'

# Find all resource limits/requests paths
kubectl get pods -A -o json | jq -c 'paths | select(. | join(".") | contains("resources"))'

# Find all paths containing "image"
kubectl get pods -A -o json | jq -c 'paths | select(. | join(".") | contains("image"))'
```

### Path-Based Queries

```bash
# Pod names
kubectl get pods -A -o json | jq -r '.items[].metadata.name'

# Pod namespaces
kubectl get pods -A -o json | jq -r '.items[].metadata.namespace'

# Pod status
kubectl get pods -A -o json | jq -r '.items[].status.phase'

# Container names
kubectl get pods -A -o json | jq -r '.items[].spec.containers[].name'

# Get values matching a path pattern
kubectl get pods -A -o json | jq -r '.items[] | paths(scalars) as $p | select($p | join(".") | contains("name")) | "\($p | join(".")): \(getpath($p))"'
```

### Conditional Path Selection

```bash
# Paths for running pods only
kubectl get pods -A -o json | jq -c '.items[] | select(.status.phase == "Running") | paths'

# Paths under spec.containers
kubectl get pods -A -o json | jq -c '.items[] | paths | select(.[0] == "spec" and .[1] == "containers")'
```

### Path Statistics and Analysis

```bash
# Count total paths
kubectl get pods -A -o json | jq '[paths] | length'

# Unique top-level keys
kubectl get pods -A -o json | jq -r 'paths | .[0]' | sort -u

# Most common root keys
kubectl get pods -A -o json | jq -r 'paths | .[0]' | sort | uniq -c | sort -nr

# Group paths by depth
kubectl get pods -A -o json | jq 'paths | group_by(length) | map({depth: .[0] | length, count: length})'

# Get parent paths only
kubectl get pods -A -o json | jq -c 'paths | .[:-1] | select(length > 0)'
```

### Other Kubernetes Resources

```bash
# Explore service paths
kubectl get services -A -o json | jq -c paths

# Explore deployment paths
kubectl get deployments -A -o json | jq -c paths

# Explore configmap paths
kubectl get configmaps -A -o json | jq -c paths
```

### Advanced jq

```bash
# Try/catch (handle missing fields)
cat data.json | jq '.items[] | {name: .name, tag: (.tags.env // "unknown")}'

# Env variables in jq
jq --arg name "$HOSTNAME" '.items[] | select(.name == $name)' data.json

# Read from file
jq '.items[].name' < data.json

# Multiple files
jq -s '.[0].items + .[1].items' file1.json file2.json

# In-place edit (with sponge)
jq '.version = "2.0"' config.json | sponge config.json

# Compact output (one line)
cat data.json | jq -c '.items[]'

# Null input (construct JSON)
jq -n '{name: "new", version: 1}'
```

### Object Operations

```bash
# to_entries — convert object to key/value array
echo '{"a":1,"b":2}' | jq 'to_entries'
# [{"key":"a","value":1},{"key":"b","value":2}]

# from_entries — convert back to object
echo '[{"key":"a","value":1}]' | jq 'from_entries'
# {"a":1}

# with_entries — transform keys/values in one pass
echo '{"name":"test","age":30}' | jq 'with_entries(.key = "prefix_" + .key)'
# {"prefix_name":"test","prefix_age":30}

# Convert JSON object to key=value lines
echo '{"host":"db","port":5432}' | jq -r 'to_entries[] | "\(.key)=\(.value)"'
# host=db
# port=5432

# Merge objects (right wins on conflicts)
echo '{"a":1}' | jq '. + {"b":2, "a":99}'
# {"a":99,"b":2}

# Check if key exists
echo '{"name":"test"}' | jq 'has("name")'
# true

# Get all keys
echo '{"a":1,"b":2,"c":3}' | jq 'keys'
# ["a","b","c"]

# Select specific fields from object
echo '{"name":"test","age":30,"tmp":"x"}' | jq '{name, age}'
# {"name":"test","age":30}
```

### Array Advanced

```bash
# Flatten nested arrays
echo '[[1,2],[3,[4,5]]]' | jq 'flatten'
# [1,2,3,4,5]

# Flatten one level only
echo '[[1,2],[3,[4,5]]]' | jq 'flatten(1)'
# [1,2,3,[4,5]]

# any/all — boolean tests on arrays
echo '[1,2,3,4,5]' | jq 'any(. > 3)'
# true
echo '[1,2,3,4,5]' | jq 'all(. > 0)'
# true

# unique_by (deduplicate by field)
echo '[{"id":1,"name":"a"},{"id":1,"name":"b"},{"id":2,"name":"c"}]' | jq 'unique_by(.id)'
# [{"id":1,"name":"a"},{"id":2,"name":"c"}]

# indices — find positions of a value
echo '["a","b","c","a","b"]' | jq 'indices("a")'
# [0,3]

# contains — check if array has all elements
echo '[1,2,3,4]' | jq 'contains([2,4])'
# true

# limit — take first N results from a generator
echo '[1,2,3,4,5]' | jq '[limit(3; .[])]'
# [1,2,3]
```

### Error Handling and Debugging

```bash
# Try — suppress errors on missing fields
echo '{"a":1}' | jq '.b.c?'
# null (no error)

# try/catch
echo '"not a number"' | jq 'try tonumber catch "invalid"'
# "invalid"

# // (alternative/null coalescing) — default when null or false
echo '{"name":null}' | jq '.name // "default"'
# "default"
echo '{}' | jq '.missing // "fallback"'
# "fallback"

# debug — print intermediate values to stderr
echo '[1,2,3]' | jq '.[] | debug | . * 2'
# stderr: ["DEBUG:",1]
# stderr: ["DEBUG:",2]
# stderr: ["DEBUG:",3]
# stdout: 2 4 6

# Access environment variables
NAME="world" jq -n 'env.NAME'
# "world"
```

### walk (Recursive Transformation)

```bash
# Recursively transform all values
# Remove newlines from all strings in entire document
cat data.json | jq 'walk(if type == "string" then gsub("\\n"; " ") else . end)'

# Recursively convert all keys to lowercase
cat data.json | jq 'walk(if type == "object" then with_entries(.key |= ascii_downcase) else . end)'

# Recursively trim all string values
cat data.json | jq 'walk(if type == "string" then ltrimstr(" ") | rtrimstr(" ") else . end)'
```

### Useful One-Liners

```bash
# Pretty-print a JSON file
jq . file.json

# Merge and deduplicate paginated API responses
jq -s 'map(.items) | flatten | unique_by(.id)' page1.json page2.json page3.json

# Convert JSON object to key=value (for .env files)
jq -r 'to_entries[] | "\(.key)=\(.value)"' config.json

# Get most recent entry from a sorted array
jq -r '.results | sort_by(.date) | reverse | .[0]' data.json

# Count items per group
jq 'group_by(.status) | map({(.[0].status): length}) | add' data.json

# Reshape — keep only specific fields
jq '[.[] | {id, name}]' data.json

# Prefix all object keys
jq 'with_entries(.key = "app_" + .key)' config.json

# Decode all base64 values in a Kubernetes secret
kubectl get secret mysecret -o json | jq '.data | map_values(@base64d)'
```

### Math and Aggregation

```bash
# Sum an array
echo '[1,2,3,4,5]' | jq 'add'
# 15

# Sum a field across objects
echo '[{"price":10},{"price":20}]' | jq '[.[].price] | add'
# 30

# Average
echo '[10,20,30]' | jq 'add / length'
# 20

# Min / Max
echo '[3,1,4,1,5]' | jq 'min'    # 1
echo '[3,1,4,1,5]' | jq 'max'    # 5

# Floor / Ceil / Round
echo '3.7' | jq 'floor'    # 3
echo '3.2' | jq 'ceil'     # 4
echo '3.5' | jq 'round'    # 4

# Absolute value
echo '-5' | jq 'fabs'      # 5

# Modulo
echo 'null' | jq '17 % 5'  # 2

# Generate a range
jq -n '[range(5)]'          # [0,1,2,3,4]
jq -n '[range(2;8;2)]'     # [2,4,6]
```

### Dates and Time

```bash
# Unix timestamp to ISO date
echo '1704067200' | jq 'todate'
# "2024-01-01T00:00:00Z"

# ISO date to Unix timestamp
echo '"2024-01-01T00:00:00Z"' | jq 'fromdateiso8601'
# 1704067200

# Current time as ISO date
jq -n 'now | todate'

# Current time as Unix timestamp
jq -n 'now'
```

### Type Selectors (Recursive Descent)

```bash
# Find all numbers anywhere in the document
echo '{"a":1,"b":{"c":2},"d":"x"}' | jq '.. | numbers'
# 1
# 2

# Find all strings
cat data.json | jq '.. | strings'

# Find all arrays
cat data.json | jq '.. | arrays'

# Find all objects
cat data.json | jq '.. | objects'

# Find all values of a specific key at any depth
cat data.json | jq '.. | .id? // empty'

# Select by type
echo '[1,"two",3,"four",null]' | jq '.[] | numbers'
# 1
# 3
```

### INDEX (Convert Array to Lookup Object)

```bash
# Create lookup by ID
echo '[{"id":"a","val":1},{"id":"b","val":2}]' | jq 'INDEX(.[]; .id)'
# {"a":{"id":"a","val":1},"b":{"id":"b","val":2}}

# Useful for merging data from two sources
jq -s '(.[0] | INDEX(.[]; .id)) as $lookup | .[1][] | . + $lookup[.id]' users.json orders.json
```

### User-Defined Functions

```bash
# Define and use a function
echo '[1,2,3]' | jq 'def double: . * 2; map(double)'
# [2,4,6]

# Function with arguments
echo '5' | jq 'def add(x): . + x; add(10)'
# 15

# Reusable formatting function
echo '[{"name":"a","age":30},{"name":"b","age":25}]' | \
    jq 'def fmt: "\(.name) is \(.age)"; .[] | fmt'
# "a is 30"
# "b is 25"
```

### Variables and Binding

```bash
# Bind a value to a variable
echo '{"price":10,"qty":3}' | jq '.price as $p | .qty * $p'
# 30

# Multiple bindings
echo '{"a":2,"b":3}' | jq '.a as $x | .b as $y | $x + $y'
# 5

# Variable from external input
jq --argjson threshold 100 '.[] | select(.value > $threshold)' data.json
```

### Path Operations

```bash
# Get path to a field
echo '{"a":{"b":{"c":1}}}' | jq 'path(.a.b.c)'
# ["a","b","c"]

# Set value at path
echo '{"a":{"b":1}}' | jq 'setpath(["a","b"]; 99)'
# {"a":{"b":99}}

# Get value at path
echo '{"a":{"b":42}}' | jq 'getpath(["a","b"])'
# 42
```

### Conditionals (Extended)

```bash
# not operator
echo '[{"active":true},{"active":false}]' | jq '.[] | select(.active | not)'
# {"active":false}

# startswith / endswith
echo '["https://a.com","http://b.com"]' | jq '.[] | select(startswith("https"))'
# "https://a.com"

echo '["file.json","file.yaml"]' | jq '.[] | select(endswith(".json"))'
# "file.json"

# empty (suppress output)
echo '[1,2,3,4,5]' | jq '.[] | if . > 3 then . else empty end'
# 4
# 5
```

### Real-World Patterns

```bash
# Merge two config files (deep merge — right wins)
jq -s '.[0] * .[1]' defaults.json overrides.json

# Remove sensitive fields before commit
jq 'del(.password, .api_key, .secret)' config.json

# Parse JSON logs — filter errors
cat app.log | jq -c 'select(.level == "error")'

# Count errors by type
cat app.log | jq -s 'group_by(.error_type) | map({type: .[0].error_type, count: length})'

# Pivot data (group then reshape)
cat data.json | jq 'group_by(.category) | map({category: .[0].category, items: map(.name)})'

# Update a specific field in config
jq '.database.host = "newhost"' config.json > config.new.json

# Rename a key
echo '{"old_name":"value"}' | jq '.new_name = .old_name | del(.old_name)'

# Convert array to lookup and join with another file
jq -s '(.[0] | INDEX(.[]; .id)) as $map | .[1][] | . + ($map[.user_id] // {})' users.json events.json

# Flatten nested structure, extract specific fields
cat data.json | jq '[.. | objects | select(has("value")) | .value]'
```

### jq Flags Reference

| Flag | Description |
|------|-------------|
| `-r` | Raw output (no quotes on strings) |
| `-c` | Compact output (one line) |
| `-s` | Slurp (read all inputs into array) |
| `-n` | Null input (don't read stdin) |
| `-e` | Exit with error if output is null/false |
| `--arg name val` | Pass string variable |
| `--argjson name val` | Pass JSON variable |
| `--slurpfile name file` | Load file into variable |
| `-f file` | Read filter from file |
| `--tab` | Use tabs for indentation |
| `--indent N` | Set indentation level |
| `-S` | Sort object keys |

## JSONPath

Used by Kubernetes (`kubectl -o jsonpath`), some REST APIs, and language-specific libraries.

### Syntax

```bash
# kubectl JSONPath syntax
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

### Key Differences from JMESPath

| Concept | JMESPath | JSONPath |
|---------|----------|----------|
| Root | (implicit) | `$` |
| Child | `.field` | `.field` or `['field']` |
| Wildcard | `[*]` | `[*]` |
| Array index | `[0]` | `[0]` |
| Array slice | `[0:5]` | `[0:5]` |
| Recursive descent | Not supported | `..` |
| Filter | `[?expr]` | `[?(@.expr)]` |
| Current element | `@` (limited) | `@` |

### kubectl JSONPath Examples

```bash
# Get all pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Get pod names (one per line using range)
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# Get name and status
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Get all pod IPs (one per line)
kubectl get pods -A -o jsonpath='{range .items[*]}{.status.podIP}{"\n"}{end}'

# CSV-like output with custom separators (namespace/name,IP)
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{","}{.status.podIP}{"\n"}{end}'

# Get specific field
kubectl get pod mypod -o jsonpath='{.spec.containers[0].image}'

# Get all container images
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Filter by condition
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'

# Get node IPs
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Custom columns (alternative to jsonpath)
kubectl get pods -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP'

# Get secret value (base64 decoded)
kubectl get secret mysecret -o jsonpath='{.data.password}' | base64 -d

# Get all images across all namespaces
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'
```

### JSONPath Recursive Descent (..)

A feature JSONPath has that JMESPath lacks — search all levels:

```bash
# Find all "name" fields at any depth
kubectl get pods -o jsonpath='{..name}'

# All container names (regardless of nesting)
kubectl get pods -o jsonpath='{..containers[*].name}'
```

### Sorting with --sort-by

`--sort-by` accepts a JSONPath expression to sort output:

```bash
# Sort nodes by name
kubectl get nodes --sort-by=.metadata.name

# Sort nodes by CPU capacity
kubectl get nodes --sort-by=.status.capacity.cpu

# Sort PVs by storage capacity
kubectl get pv --sort-by=.spec.capacity.storage

# Sort pods by creation time
kubectl get pods --sort-by=.metadata.creationTimestamp

# Sort pods by restart count
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'
```

### Custom Columns

A cleaner alternative to JSONPath range for tabular output. No need for `items` — custom-columns iterates automatically:

```bash
# Node name and CPU
kubectl get nodes -o custom-columns=NODE:.metadata.name,CPU:.status.capacity.cpu

# Node name, OS, kernel, container runtime
kubectl get nodes -o custom-columns=\
NODE:.metadata.name,\
OS:.status.nodeInfo.osImage,\
KERNEL:.status.nodeInfo.kernelVersion,\
RUNTIME:.status.nodeInfo.containerRuntimeVersion

# Node name and internal IP
kubectl get nodes -o custom-columns=NODE:.metadata.name,IP:.status.addresses[0].address

# Node taints
kubectl get nodes -o custom-columns=NODE:.metadata.name,TAINT:.spec.taints[*].effect,KEY:.spec.taints[*].key

# PVs with name and capacity (combined with --sort-by)
kubectl get pv -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage \
    --sort-by=.spec.capacity.storage

# Pods with name, image, and status
kubectl get pods -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image,STATUS:.status.phase
```

### Filter by Condition (Status)

Use `?(@.type=="value")` to filter within arrays of conditions:

```bash
# Nodes with their Ready status
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Node name, taint effect, and Ready status
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints[*].effect}{"\t"}{.status.conditions[?(@.type=="Ready")].type}{"\n"}{end}'

# Pods that are not Ready
kubectl get pods -o jsonpath='{.items[?(@.status.phase!="Running")].metadata.name}'
```

### JSONPath in Other Tools

```bash
# Python (jsonpath-ng)
pip install jsonpath-ng

# Node.js (jsonpath)
npm install jsonpath

# Go (gjson — not JSONPath but similar)
# Used in many Go tools for path-based JSON access
```

## When to Use Which

| Scenario | Best Tool | Why |
|----------|-----------|-----|
| AWS CLI output | JMESPath | Built-in, no extra tools |
| Complex JSON transformation | jq | Most powerful, full programming |
| Kubernetes resources | JSONPath | Built into kubectl |
| Shell scripts (simple extraction) | jq | Available everywhere, handles edge cases |
| CI/CD pipelines | jq | Portable, scriptable |
| Creating new JSON | jq | Only tool that can construct data |
| REST API response parsing | jq or JSONPath | Depends on the framework |
| Log file processing (JSON logs) | jq | Stream processing, filters |
| Node.js / browser environments | JavaScript | Native language, full flexibility |
| Structured APIs with schema enforcement | GraphQL | Type-safe, client-driven queries |
| XML documents | XPath | Hierarchical path notation for XML |
| Simple path queries, max compatibility | JSONPath | Easy, widely supported |
| Complex filtering, language-agnostic | JMESPath | Single spec, consistent |

## Side-by-Side Examples

### Sample JSON Data

All examples below use this structure:

```json
{
  "store": {
    "name": "TechBooks",
    "books": [
      { "id": 1, "title": "JavaScript Guide", "price": 25.99, "author": "John Doe" },
      { "id": 2, "title": "Python Basics", "price": 19.99, "author": "Jane Smith" },
      { "id": 3, "title": "Web Design", "price": 35.50, "author": "Bob Johnson" }
    ]
  }
}
```

### Select a field

```bash
# JMESPath
--query 'Instances[0].InstanceId'

# jq
jq '.Instances[0].InstanceId'

# JSONPath
{.Instances[0].InstanceId}
```

### Filter by value

```bash
# JMESPath
--query 'Items[?status==`active`].name'

# jq
jq '.Items[] | select(.status == "active") | .name'

# JSONPath
{.Items[?(@.status=="active")].name}
```

### Get multiple fields

```bash
# JMESPath
--query 'Items[].{Name:name, ID:id}'

# jq
jq '.Items[] | {Name: .name, ID: .id}'

# JSONPath (limited — no rename)
{.Items[*].name} {.Items[*].id}
```

### Sort by field

```bash
# JMESPath
--query 'sort_by(Items, &date)'

# jq
jq '.Items | sort_by(.date)'

# JSONPath
# Not supported — no sorting capability
```

### Count elements

```bash
# JMESPath
--query 'length(Items)'

# jq
jq '.Items | length'

# JSONPath
# Not supported — no functions
```

### Flatten nested arrays

```bash
# JMESPath
--query 'Outer[].Inner[].field'

# jq
jq '.Outer[].Inner[].field'

# JSONPath
{.Outer[*].Inner[*].field}
```

### First non-null value

```bash
# JMESPath
--query 'not_null(PublicIP, PrivateIP)'

# jq
jq '.PublicIP // .PrivateIP'

# JSONPath
# Not supported
```

## JavaScript (Native)

When you're already in Node.js or browser environments, native JavaScript is the most direct approach — no query language needed.

```javascript
const data = { /* loaded JSON */ };

// Simple access
data.store.name                          // "TechBooks"
data.store.books[0].title                // "JavaScript Guide"

// Map all titles
data.store.books.map(b => b.title)
// ["JavaScript Guide", "Python Basics", "Web Design"]

// Filter and map
data.store.books
  .filter(b => b.price < 30)
  .map(b => b.title)
// ["JavaScript Guide", "Python Basics"]

// Sort by price
data.store.books
  .sort((a, b) => a.price - b.price)
  .map(b => ({ title: b.title, price: b.price }))
// [{title: "Python Basics", price: 19.99}, {title: "JavaScript Guide", price: 25.99}, ...]

// Find single item
data.store.books.find(b => b.id === 2)

// Reduce (total price)
data.store.books.reduce((sum, b) => sum + b.price, 0)
// 81.48

// Destructuring
const { store: { books } } = data;
const titles = books.map(({ title }) => title);
```

### When to Use JavaScript

- Already in Node.js/browser — no extra dependencies
- Need complex custom logic (loops, conditionals, regex)
- Performance-critical (native execution, no parsing overhead)
- Building application logic around the data (not just querying)

## Extended Feature Comparison

| Feature | JSONPath | JMESPath | JavaScript | jq |
|---------|----------|----------|------------|-----|
| **Syntax style** | Path-based (`$`) | Expression-based | Native language | Functional/pipeline |
| **Filtering** | Basic (`?(@.field)`) | Rich (`?field < value`) | Full language | `select()` |
| **Functions** | Minimal | Many built-in | Unlimited | Many built-in |
| **Sorting** | Not built-in | `sort_by()` | `.sort()` | `sort_by()` |
| **Grouping** | No | No | `.reduce()` | `group_by()` |
| **Create new JSON** | No | Limited (multi-select hash) | Yes | Yes |
| **Regex** | No | No | Yes | `test()`, `match()` |
| **Arithmetic** | No | No | Yes | Yes |
| **Recursive descent** | Yes (`..`) | No | Manual | `..` (recurse) |
| **String functions** | No | `contains`, `starts_with` | Full | `split`, `join`, `gsub` |
| **Implementation** | Multiple variants | Single spec | Standard | Single tool |
| **Learning curve** | Easy | Medium | Variable | Medium |
| **Use context** | Config files, kubectl | AWS CLI, APIs | Runtime code | CLI/scripting |

## Practical Considerations

**Choose JSONPath if:**
- You need simple path-based queries
- You want maximum compatibility across tools and languages
- You're working with Kubernetes (`kubectl`)
- Your queries are straightforward (get field X from object Y)

**Choose JMESPath if:**
- You need complex filtering and transformations
- You want a consistent, unambiguous specification
- You're working with AWS CLI or tools that support it
- You need built-in functions (sort, max, length, join)

**Choose JavaScript if:**
- You're already in a Node.js/browser environment
- You need complex custom logic beyond simple queries
- Performance is critical and you want native execution
- You need to combine querying with application logic

**Choose jq if:**
- You're working in the command line or shell scripts
- You need powerful transformations in a Unix pipeline
- You're processing JSON logs or API responses in CI/CD
- You need to create, merge, or restructure JSON

## Other Tools

### gron (Make JSON greppable)

```bash
# Install
brew install gron
apt install gron

# Flatten JSON to assignment statements
echo '{"name":"test","items":[1,2,3]}' | gron
# json = {};
# json.name = "test";
# json.items = [];
# json.items[0] = 1;
# json.items[1] = 2;
# json.items[2] = 3;

# Grep then unflatten
aws ec2 describe-instances --output json | gron | grep InstanceId | gron --ungron
```

### fx (Interactive JSON viewer)

```bash
# Install
npm install -g fx

# Interactive browsing
cat data.json | fx

# With expression
cat data.json | fx '.items.length'
```

### yq (YAML/JSON processor — jq wrapper)

```bash
# Install
brew install yq
pip install yq

# Process YAML files (same syntax as jq)
yq '.metadata.name' pod.yaml

# Convert YAML to JSON
yq -o json '.' config.yaml

# Convert JSON to YAML
cat data.json | yq -P '.'
```

### jp (JMESPath CLI)

```bash
# Install
go install github.com/jmespath/jp/cmd/jp@latest
pip install jmespath-terminal

# Test JMESPath expressions
echo '{"items":[{"name":"a"},{"name":"b"}]}' | jp 'items[].name'

# Pipe AWS output for testing
aws ec2 describe-instances --output json | jp 'Reservations[].Instances[].InstanceId'
```

### dasel (Data selector — JSON, YAML, TOML, CSV)

```bash
# Install
brew install dasel

# Query JSON
echo '{"name":"test"}' | dasel -r json '.name'

# Query YAML
dasel -f config.yaml '.metadata.name'

# Convert between formats
dasel -f input.json -r json -w yaml > output.yaml
```

## Performance Comparison

| Operation | JMESPath (AWS) | jq | JSONPath (kubectl) |
|-----------|---------------|----|--------------------|
| Simple field access | Fast (server-side with `--filter`) | Fast | Fast |
| Large dataset filtering | Moderate (client-side) | Fast (streaming) | Fast |
| Complex transformation | N/A (limited) | Fast | N/A (limited) |
| Memory usage | Low | Low-moderate | Low |
| Startup time | None (built-in) | ~10ms | None (built-in) |

For large JSON files (>100MB), jq's streaming mode (`--stream`) handles data without loading it all into memory.

```bash
# Stream processing for large files
jq --stream 'select(.[0][-1] == "InstanceId") | .[1]' large-output.json
```
