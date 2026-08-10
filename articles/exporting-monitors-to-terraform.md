<img src="/articles/images/datadog-logo.svg" alt="bash logo" width="150"/>

# Exporting Datadog Monitors to Terraform

Managing Datadog monitors as code gives you version control, peer review, and reproducibility. This guide covers three ways to export existing monitors into Terraform HCL.

## Method 1: Datadog Console (Easiest)

The Datadog web UI has a built-in Terraform export:

1. Go to **Monitors > Manage Monitors**
2. Click on the monitor you want to export
3. Click the **Export** button (top-right)
4. Select **Terraform**
5. Copy the generated HCL into your `.tf` file

This gives you a complete, ready-to-use resource block. Best for exporting a few monitors at a time.

## Method 2: API Export + jq Conversion

For bulk export, pull all monitors via the API and convert to HCL with `jq`.

### Step 1: Export monitors from the API

```bash
export DD_API_KEY="your-api-key"
export DD_APPLICATION_KEY="your-app-key"

curl -s -X GET "https://api.datadoghq.com/api/v1/monitor" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APPLICATION_KEY}" \
  -H "Content-Type: application/json" > monitors_export.json
```

> **Note:** If your account is on the EU site, use `https://api.datadoghq.eu` instead.

### Step 2: Format and inspect

```bash
jq . monitors_export.json | less
```

### Step 3: Convert to Terraform HCL

```bash
jq -r '.[] | "resource \"datadog_monitor\" \"\(.name | gsub("[^a-zA-Z0-9]"; "_"))\" {\n  name = \"\(.name)\"\n  type = \"\(.type)\"\n  query = <<EOT\n\(.query)\nEOT\n}\n"' monitors_export.json
```

This generates a skeleton resource block for each monitor. You'll want to add:

- `message` (notification body)
- `tags`
- `monitor_thresholds`
- Any other settings (priority, notify_no_data, etc.)

### Step 4: Import into Terraform state

After adding the HCL to your `.tf` files, import each monitor so Terraform knows it already exists:

```bash
terraform import datadog_monitor.Monitor_Name <monitor-id>
```

You can find the monitor ID in the URL when viewing it in Datadog (`/monitors/<id>`), or from the JSON export:

```bash
jq -r '.[] | "\(.id) \(.name)"' monitors_export.json
```

## Tips

- **Header format matters:** HTTP headers use dashes (`DD-API-KEY`), environment variables use underscores (`DD_API_KEY`).
- **No spaces around `=` in exports:** `export DD_API_KEY="value"` not `export DD_API_KEY = "value"`.
- **Run `terraform plan` after import** to verify no drift between your HCL and the live monitor.
- **Use the console export for accuracy** — it includes all fields. The jq method gives you a starting skeleton that needs manual enrichment.

## Region-Specific API URLs

| Site | API URL |
|------|---------|
| US1 (default) | `https://api.datadoghq.com` |
| US3 | `https://api.us3.datadoghq.com` |
| US5 | `https://api.us5.datadoghq.com` |
| EU | `https://api.datadoghq.eu` |
| AP1 | `https://api.ap1.datadoghq.com` |
