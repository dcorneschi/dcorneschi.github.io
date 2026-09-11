# Understanding <=, +, and Other Signs in a Terraform Plan

When you run `terraform plan` or `terraform apply`, Terraform prints a summary of every change it intends to make. Each resource and attribute is prefixed with a symbol that tells you *what kind* of action Terraform will take. Reading these symbols correctly is the difference between confidently approving a plan and accidentally destroying production.

This guide covers every symbol you'll see, what it means, and the gotchas behind the ones that surprise people — especially `<=`.

## The Action Symbols at a Glance

| Symbol | Action | Meaning |
|--------|--------|---------|
| `+` | **create** | A new resource or attribute will be added |
| `-` | **destroy** | The resource or attribute will be removed |
| `~` | **update in place** | An existing attribute changes without recreating the resource |
| `-/+` | **destroy and recreate** | The resource is destroyed, then a new one is created (replacement) |
| `+/-` | **create then destroy** | Same as above but with `create_before_destroy = true` |
| `<=` | **read** | A data source will be read during apply (not a change to infrastructure) |
| `#` | comment / note | Informational lines, e.g. why a resource is being replaced |
| (no symbol) | no change | The attribute is shown for context but is unchanged |

The final line of a plan aggregates these:

```text
Plan: 3 to add, 1 to change, 2 to destroy.
```

## + — Create

A `+` means the object does not exist yet and Terraform will create it. You see it for brand-new resources and for individual attributes being set.

```hcl
  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami                          = "ami-0abcdef1234567890"
      + instance_type                = "t3.micro"
      + id                           = (known after apply)
      + private_ip                   = (known after apply)
    }
```

- Every attribute of a new resource is prefixed with `+`.
- `(known after apply)` means the value isn't decidable until the resource actually exists (IDs, computed IPs, ARNs).

## - — Destroy

A `-` means the object will be removed. This appears when you delete a resource block, remove it from `count`/`for_each`, or when a whole resource is going away.

```hcl
  # aws_instance.old will be destroyed
  - resource "aws_instance" "old" {
      - ami           = "ami-0abcdef1234567890" -> null
      - instance_type = "t3.micro" -> null
      - id            = "i-0123456789abcdef0" -> null
    }
```

Destroys are the highest-risk action. The `# ... will be destroyed` comment line above the block always tells you exactly which address is affected. Read those lines carefully before approving.

## ~ — Update in Place

A `~` means the resource stays but one or more attributes change. Terraform shows the transition as `old -> new`.

```hcl
  # aws_instance.web will be updated in-place
  ~ resource "aws_instance" "web" {
        id            = "i-0123456789abcdef0"
      ~ instance_type = "t3.micro" -> "t3.large"
        # (other attributes unchanged)
    }
```

- Only the changed attributes carry `~`; unchanged ones are shown without a symbol for context.
- Whether an attribute can be updated in place (`~`) versus forcing a replacement (`-/+`) is defined by the provider.

## -/+ and +/- — Replacement (Destroy and Recreate)

Some attributes cannot be changed on a live resource, so Terraform must destroy the old one and create a new one. This is a **replacement**, shown as `-/+`.

```hcl
  # aws_instance.web must be replaced
-/+ resource "aws_instance" "web" {
      ~ ami           = "ami-old" -> "ami-new" # forces replacement
      ~ id            = "i-0123456789abcdef0" -> (known after apply)
        instance_type = "t3.micro"
    }
```

Two critical details:

- The `# forces replacement` comment marks the exact attribute triggering the recreate.
- The order matters:
  - `-/+` → **destroy first, then create** (default). There will be downtime.
  - `+/-` → **create first, then destroy** — happens when the resource has `lifecycle { create_before_destroy = true }`.

## <= — Read (Data Sources)

This is the symbol that confuses most people. `<=` does **not** mean "less than or equal to." It represents a **data source read** that Terraform will perform during apply rather than at plan time.

```hcl
  # data.aws_ami.ubuntu will be read during apply
 <= data "aws_ami" "ubuntu" {
      + id           = (known after apply)
      + most_recent  = true
      + owners       = ["099720109477"]
    }
```

Key points about `<=`:

- It applies to **`data` blocks**, never to `resource` blocks.
- It is **not a change to your infrastructure** — Terraform is only fetching information. Nothing is created, modified, or destroyed.
- It does **not** appear in the `Plan: X to add, Y to change, Z to destroy` count, because reading isn't a mutating action.

### Why does a data source read defer to apply time?

Normally Terraform reads data sources during the plan (refresh) phase and knows their values immediately. You see `<=` (deferred to apply) when the data source's arguments depend on values that are themselves `(known after apply)`.

```hcl
data "aws_instance" "web" {
  # This depends on an instance that doesn't exist yet,
  # so the read is deferred until apply.
  instance_id = aws_instance.web.id
}
```

Because `aws_instance.web.id` won't be known until the instance is created during apply, Terraform can't read the data source at plan time — hence the `<=` deferral.

## No Symbol — Context Only

Lines with no leading symbol are unchanged attributes displayed to give you context around the ones that *are* changing. Terraform collapses long runs of these into a note:

```hcl
        # (7 unchanged attributes hidden)
```

Use `terraform plan -no-color` and scroll, or `terraform show` on a saved plan, if you want to expand hidden attributes.

## # — Comment Lines

Lines beginning with `#` are human-readable annotations, not actions. The most important ones:

| Comment | Meaning |
|---------|---------|
| `# ... will be created` | Precedes a `+` resource block |
| `# ... will be destroyed` | Precedes a `-` resource block |
| `# ... will be updated in-place` | Precedes a `~` resource block |
| `# ... must be replaced` | Precedes a `-/+` resource block |
| `# ... will be read during apply` | Precedes a `<=` data block |
| `# forces replacement` | Marks the attribute causing a replacement |
| `# (N unchanged attributes hidden)` | Collapsed context |

## Reading the Full Example

```hcl
Terraform will perform the following actions:

  # aws_instance.new will be created
  + resource "aws_instance" "new" {
      + ami           = "ami-123"
      + instance_type = "t3.micro"
      + id            = (known after apply)
    }

  # aws_instance.web will be updated in-place
  ~ resource "aws_instance" "web" {
        id            = "i-abc"
      ~ instance_type = "t3.micro" -> "t3.small"
    }

  # aws_instance.legacy must be replaced
-/+ resource "aws_instance" "legacy" {
      ~ ami = "ami-old" -> "ami-new" # forces replacement
    }

  # aws_instance.retired will be destroyed
  - resource "aws_instance" "retired" {
      - id = "i-xyz" -> null
    }

  # data.aws_ami.ubuntu will be read during apply
 <= data "aws_ami" "ubuntu" {
      + id          = (known after apply)
      + most_recent = true
    }

Plan: 2 to add, 1 to change, 2 to destroy.
```

Note that the `-/+` replacement counts as **both** 1 add and 1 destroy in the summary, while the `<=` data read is not counted at all.

## Machine-Readable Plans

For scripting and CI, don't parse the symbols from text — use JSON output instead.

```bash
# Save the plan, then convert to JSON
terraform plan -out=tfplan
terraform show -json tfplan > plan.json

# Extract each resource's actions with jq
jq '.resource_changes[] | {address, actions: .change.actions}' plan.json
```

The `actions` array maps directly to the symbols:

| JSON `actions` | Plan symbol |
|----------------|-------------|
| `["create"]` | `+` |
| `["delete"]` | `-` |
| `["update"]` | `~` |
| `["delete", "create"]` | `-/+` |
| `["create", "delete"]` | `+/-` |
| `["read"]` | `<=` |
| `["no-op"]` | (no symbol) |

## Quick Reference

| You see | Terraform will | Risk |
|---------|----------------|------|
| `+` | Create something new | Low |
| `~` | Modify in place | Low–Medium |
| `<=` | Read a data source (no change) | None |
| `-/+` | Destroy then recreate | High (downtime) |
| `+/-` | Create then destroy | Medium |
| `-` | Destroy | High |

## Key Takeaways

- `<=` is a **data source read**, not a comparison operator — it changes nothing and isn't counted in the plan summary.
- `+` and `-` are create and destroy; `~` is an in-place update.
- `-/+` (and its `create_before_destroy` sibling `+/-`) is a **replacement** — the riskiest common action, always flagged with `# forces replacement`.
- Always read the `# ... will be destroyed` and `# ... must be replaced` comment lines before approving.
- For automation, parse `terraform show -json` rather than the human-readable symbols.
