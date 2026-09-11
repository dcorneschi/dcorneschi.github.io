# Deleting GitHub Deployments with the gh CLI

GitHub records a deployment entry for every deploy to an environment, and these pile up over time. There's no UI button to bulk-remove them, but the REST API (via the `gh` CLI) can. The catch: an **active** deployment can't be deleted directly — you must mark it `inactive` first. This guide covers listing, deleting one, and a script to clear them in bulk.

## Prerequisites

- [GitHub CLI](https://cli.github.com/) installed and authenticated: `gh auth login`.
- `jq` for parsing JSON responses.
- Token scope with repo access (the deployments API needs `repo_deployment` or `repo`).

## List Deployments

```bash
REPO="OWNER/REPO"

gh api "repos/$REPO/deployments?per_page=100" | jq -c '.[]' | while read -r DEPLOYMENT; do
  ID=$(echo "$DEPLOYMENT" | jq -r '.id')
  ENV=$(echo "$DEPLOYMENT" | jq -r '.environment')
  SHA=$(echo "$DEPLOYMENT" | jq -r '.sha')
  STATUS=$(gh api "repos/$REPO/deployments/$ID/statuses" | jq -r '.[0].state')
  echo "ID: $ID | SHA: $SHA | Status: $STATUS | Env: $ENV"
done
```

Each deployment's current state comes from its most recent status entry, which is why the loop makes a second call per deployment.

## Delete a Single Deployment

Two steps — set it inactive, then delete:

```bash
REPO="OWNER/REPO"
ID="DEPLOYMENT_ID"

# 1. Mark inactive (active deployments can't be deleted)
gh api --method POST \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  "repos/$REPO/deployments/$ID/statuses" \
  -f 'state=inactive'

# 2. Delete it
gh api --method DELETE \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  "repos/$REPO/deployments/$ID"
```

## Bulk-Delete Script

Deletes all deployments in a repo, optionally scoped to one environment, with a confirmation prompt.

```bash
#!/bin/bash
# delete-github-deployments.sh
# Usage: ./delete-github-deployments.sh OWNER/REPO [ENVIRONMENT]
set -euo pipefail

REPO="${1:?Usage: $0 OWNER/REPO [ENVIRONMENT]}"
ENV="${2:-}"

if [ -z "$ENV" ]; then
  API_URL="repos/$REPO/deployments?per_page=100"
else
  API_URL="repos/$REPO/deployments?per_page=100&environment=${ENV// /%20}"
fi

echo "Fetching deployments for $REPO..."
IDS=$(gh api "$API_URL" | jq -r '.[].id')
COUNT=$(echo "$IDS" | grep -c . || true)

if [ -z "$IDS" ] || [ "$COUNT" -eq 0 ]; then
  echo "No deployments found."
  exit 0
fi

echo "Found $COUNT deployment(s)."
read -p "Mark as inactive and delete all? (y/n): " confirm
[ "$confirm" = "y" ] || { echo "Cancelled."; exit 0; }

for ID in $IDS; do
  echo "Processing deployment $ID..."
  # Mark inactive first
  gh api --method POST \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    "repos/$REPO/deployments/$ID/statuses" \
    -f 'state=inactive' > /dev/null 2>&1
  # Then delete
  gh api --method DELETE \
    -H "Accept: application/vnd.github+json" \
    -H "X-GitHub-Api-Version: 2022-11-28" \
    "repos/$REPO/deployments/$ID" > /dev/null 2>&1
  echo "  deleted $ID"
done

echo "Done. Deleted $COUNT deployment(s)."
```

### Usage

```bash
# All deployments in the repo
./delete-github-deployments.sh myuser/myrepo

# Only a specific environment
./delete-github-deployments.sh myuser/myrepo production
```

## Notes and Gotchas

- **Inactive-first is mandatory.** The API rejects deleting an active deployment; the `state=inactive` status POST clears that.
- **Pagination caps at 100.** `per_page=100` is the max per request — with more than 100 deployments, re-run the script (each run removes up to a page) or add page iteration.
- **This removes deployment *entries*, not the environment.** The environment (and its protection rules/secrets) stays; only the deployment history is cleared.
- **It's irreversible.** Deleted deployments and their statuses are gone. Since deployment records are often referenced for audit/history, confirm you want them removed.
- Silencing output with `> /dev/null 2>&1` also hides errors — drop it while testing so you can see any API failures (rate limits, permission issues).

## Related

- [GitHub CLI (gh) Cheatsheet](articles/gh-cli-cheatsheet.md)
