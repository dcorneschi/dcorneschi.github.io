# Importing a Datadog Dashboard from JSON

Datadog dashboards are portable as JSON, so you can move one between orgs, restore it from version control, or generate it from a template. This guide covers importing dashboard JSON through the UI, the API, a reusable script, and Terraform, plus validation, backup, and troubleshooting.

For the broader dashboard and API context, see the [Datadog Dashboards Guide](articles/datadog-dashboards-guide.md) and the [Datadog API Reference](articles/datadog-api-reference.md).

## Pick Your Datadog Site First

Datadog runs on multiple sites (US1, US3, US5, EU, etc.), each with its own API host. Using the wrong host causes authentication or 404 errors. Set it once:

```bash
# US1 (default). Others: us3.datadoghq.com, us5.datadoghq.com, datadoghq.eu, ap1.datadoghq.com
export DD_SITE="https://api.datadoghq.com"
export DD_API_KEY="your-api-key"
export DD_APP_KEY="your-application-key"
```

The examples below use `${DD_SITE}` so they work on any site. The web UI equivalent lives at `https://app.datadoghq.com` (or `app.datadoghq.eu`, etc.).

## Method 1: Import via the UI

The quickest path for a one-off import.

1. Log in to the Datadog UI.
2. Go to **Dashboards** and create a **New Dashboard**.
3. Open the dashboard settings (gear/cog menu, top right).
4. Choose **Import dashboard JSON**.
5. Paste the JSON, then save.

Then verify: widgets render, metrics show data (allow a few minutes), and template variables (e.g. cluster, ASG) filter correctly.

If your JSON lives inside a Markdown reference file between ` ```json ` fences, extract it first:

```bash
sed -n '/^```json/,/^```/p' dashboard-reference.md | sed '1d;$d' > dashboard.json
```

## Method 2: Import via the API

Best for automation and repeatable imports. A **POST** creates a new dashboard; Datadog assigns the ID.

```bash
curl -X POST "${DD_SITE}/api/v1/dashboard" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -d @dashboard.json
```

A successful response includes the assigned `id` and `url`:

```json
{
  "id": "abc-123-def",
  "title": "EKS Cluster Autoscaler Dashboard",
  "url": "/dashboard/abc-123-def/eks-cluster-autoscaler-dashboard"
}
```

> **Correction on "custom dashboard IDs":** you cannot choose the ID on creation. `POST` always mints a new ID. `PUT /api/v1/dashboard/{id}` **updates an existing** dashboard identified by its server-assigned ID — it does not create one at an arbitrary ID you pick. Import once with POST, note the returned `id`, then use PUT to update that same dashboard thereafter.

### Update an existing dashboard

```bash
DASHBOARD_ID="abc-123-def"

curl -X PUT "${DD_SITE}/api/v1/dashboard/${DASHBOARD_ID}" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -d @dashboard.json
```

### One-line extract-and-import

```bash
sed -n '/^```json/,/^```/p' dashboard-reference.md | sed '1d;$d' \
  | curl -X POST "${DD_SITE}/api/v1/dashboard" \
      -H "Content-Type: application/json" \
      -H "DD-API-KEY: ${DD_API_KEY}" \
      -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
      -d @-
```

## Method 3: A Reusable Import Script

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${DD_API_KEY:?set DD_API_KEY}"
: "${DD_APP_KEY:?set DD_APP_KEY}"
DD_SITE="${DD_SITE:-https://api.datadoghq.com}"

file="${1:?usage: import-dashboard.sh <dashboard.json>}"
[ -f "$file" ] || { echo "File not found: $file" >&2; exit 1; }

# Validate JSON before sending
jq empty "$file" || { echo "Invalid JSON: $file" >&2; exit 1; }

response="$(curl -sS -X POST "${DD_SITE}/api/v1/dashboard" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -d @"$file")"

id="$(printf '%s' "$response" | jq -r '.id // empty')"
url="$(printf '%s' "$response" | jq -r '.url // empty')"

if [ -n "$id" ]; then
  echo "Imported: $id"
  echo "URL: https://app.datadoghq.com${url}"
else
  echo "Import failed:" >&2
  printf '%s\n' "$response" >&2
  exit 1
fi
```

Using `jq` for parsing (rather than `grep`/`cut`) is more robust against formatting changes, and `set -euo pipefail` plus the `: "${VAR:?}"` guards fail fast on missing keys or a bad file.

## Method 4: Manage It with Terraform

For infrastructure-as-code, the `datadog_dashboard_json` resource stores the raw JSON and reconciles it.

```hcl
# versions.tf
terraform {
  required_providers {
    datadog = {
      source  = "DataDog/datadog"
      version = "~> 3.0"
    }
  }
}

provider "datadog" {
  api_key  = var.datadog_api_key
  app_key  = var.datadog_app_key
  api_url  = var.datadog_api_url   # set per site, e.g. https://api.datadoghq.eu/
}
```

```hcl
# dashboard.tf
resource "datadog_dashboard_json" "cluster_autoscaler" {
  dashboard = file("${path.module}/dashboard.json")
}

output "dashboard_url" {
  value = datadog_dashboard_json.cluster_autoscaler.url
}
```

```hcl
# variables.tf
variable "datadog_api_key" {
  type      = string
  sensitive = true
}
variable "datadog_app_key" {
  type      = string
  sensitive = true
}
variable "datadog_api_url" {
  type    = string
  default = "https://api.datadoghq.com/"
}
```

Provide the keys via environment variables (`TF_VAR_datadog_api_key`, `TF_VAR_datadog_app_key`) or a secrets manager rather than a committed `terraform.tfvars`. Then:

```bash
terraform init
terraform plan
terraform apply
```

Terraform will now detect drift if the dashboard is edited in the UI, which is the main advantage over one-shot API imports. See the [Terraform Variables Guide](articles/terraform-variables-guide.md) for handling the sensitive keys.

## Validate Before Importing

```bash
# jq (fails non-zero on invalid JSON)
jq empty dashboard.json && echo "valid" || echo "invalid"

# Python
python3 -m json.tool dashboard.json > /dev/null && echo "valid" || echo "invalid"
```

A local syntax check catches the most common failure (trailing commas, unescaped quotes) before an API round-trip.

## List, Back Up, and Delete

### List dashboards

```bash
curl -s "${DD_SITE}/api/v1/dashboard" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  | jq -r '.dashboards[] | "\(.id)\t\(.title)"'
```

### Back up all dashboards

```bash
#!/usr/bin/env bash
set -euo pipefail
DD_SITE="${DD_SITE:-https://api.datadoghq.com}"
out="datadog-dashboards-$(date +%F)"
mkdir -p "$out"

curl -s "${DD_SITE}/api/v1/dashboard" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  | jq -r '.dashboards[].id' \
  | while read -r id; do
      echo "Backing up $id"
      curl -s "${DD_SITE}/api/v1/dashboard/${id}" \
        -H "DD-API-KEY: ${DD_API_KEY}" \
        -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
        | jq '.' > "${out}/${id}.json"
    done
echo "Saved to ${out}/"
```

### Delete a dashboard

```bash
curl -X DELETE "${DD_SITE}/api/v1/dashboard/${DASHBOARD_ID}" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}"
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `403 Forbidden` / `Forbidden` | Wrong site, or missing/invalid app key | Confirm `DD_SITE` matches your org; app key must be valid and active |
| `400` invalid JSON | Malformed body | `jq empty dashboard.json` before importing |
| Import works, widgets show "No Data" | Metrics not present, or wrong tags | Check Metrics Explorer; confirm the Agent/integration emits those metrics |
| Template variables don't filter | Tag keys don't match your environment | Align variable prefixes with your actual tag keys |
| Cloud metrics missing (e.g. ASG) | Cloud integration not enabled or under-scoped | Enable the AWS/Azure/GCP integration with the required read permissions |

Verbose request for debugging a failed import:

```bash
curl -v -X POST "${DD_SITE}/api/v1/dashboard" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  --data-binary @dashboard.json
```

## Best Practices

- Keep dashboard JSON in version control; treat the UI as a preview, Git as the source of truth.
- Set `DD_SITE` explicitly so scripts work across US/EU/other sites.
- Validate JSON locally before importing.
- Prefer Terraform (`datadog_dashboard_json`) when you want drift detection and review.
- Never commit API/app keys; use environment variables or a secrets manager.
- Back up dashboards on a schedule before bulk changes.
- Test imports in a non-production org first when possible.

## Quick Reference

```bash
# Extract JSON from a Markdown reference file
sed -n '/^```json/,/^```/p' dashboard-reference.md | sed '1d;$d' > dashboard.json

# Validate
jq empty dashboard.json && echo valid

# Create (POST — new dashboard, Datadog assigns the ID)
curl -X POST "${DD_SITE}/api/v1/dashboard" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: ${DD_API_KEY}" -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -d @dashboard.json

# Update (PUT — existing dashboard by its ID)
curl -X PUT "${DD_SITE}/api/v1/dashboard/${DASHBOARD_ID}" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: ${DD_API_KEY}" -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -d @dashboard.json
```

For related material, see the [Datadog Dashboards Guide](articles/datadog-dashboards-guide.md), the [Datadog API Reference](articles/datadog-api-reference.md), and the [Datadog Agent Cheatsheet](articles/datadog-agent-cheatsheet.md).
