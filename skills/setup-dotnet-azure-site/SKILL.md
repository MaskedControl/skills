---
name: setup-dotnet-azure-site
description: >
  Use when creating a new Azure deployment environment (staging, QA, demo, testing) for an
  existing .NET production app hosted on Azure App Service.
  Triggers on: "set up a staging environment", "create a new deployment site", "I need a testing
  environment that mirrors production", "spin up a QA site." Covers Azure resource provisioning
  via az CLI, GitHub branch creation, GitHub Actions CI/CD (detecting existing workflows or
  creating fresh), and per-environment secret isolation via GitHub Environments. Skip for non-.NET
  apps (Python, Node.js) or non-Azure hosting (Vercel, AWS, GCP).
license: Complete terms in LICENSE (MIT)
---

# Setup .NET Azure Deployment Site

## Overview

Creates a complete new deployment environment (staging, QA, demo) for an existing .NET
Azure-hosted app by analyzing production, provisioning the right Azure resources, detecting or
creating GitHub Actions CI/CD, and wiring up per-environment secrets.

**Core principle:** Share what can be shared; skip what staging doesn't need. New databases can
live on the existing production SQL server — no new server cost. Communication and email services
can be referenced cross-resource-group at no extra cost. Data Factory and alert rules are almost
never needed in staging.

## Prerequisites

Before starting, confirm:
- `az login` — Azure CLI authenticated
- `gh auth login` — GitHub CLI authenticated
- Contributor access on the Azure subscription
- Admin access on the GitHub repository

## Phase 1: Gather Information

Ask the user for these inputs upfront:

| Input | Example | Used for |
|-------|---------|----------|
| Environment name | `staging`, `qa`, `demo` | Naming all resources |
| Production resource group | `myapp-prod-rg` | Analysis source |
| GitHub repo | `owner/repo` | Branch + workflow |
| Base branch | `main`, `master` | Starting point for new branch |
| New branch name | `staging` | CI/CD trigger (must match GitHub Environment name exactly) |
| Azure region | `australiaeast`, `eastus` | Resource location |
| App Service Plan tier | `B1` (default), `B2`, `S1` | Cost vs performance |
| .NET version | `8`, `9` | Runtime for App Service and Function App |
| Database strategy | copy+scrub or empty+migrations | Determines Phase 4 database steps |

Derive resource names from environment name (confirm with user):
- Resource group: `{product}-{env}-rg`
- App prefix: `{product}-{env}`
- App URL: `{product}-{env}.azurewebsites.net`

## Phase 2: Analyze Production

```bash
az resource list \
  --resource-group <prod-rg> \
  --query "[].{name:name, type:type, kind:kind}" \
  --output table
```

## Phase 3: Resource Decision Matrix

For each resource type found in production, apply:

| Resource Type | Decision | Reason |
|---------------|----------|--------|
| App Service Plan (`Microsoft.Web/serverFarms`) | **Create new** (B1 tier) | Staging needs less scale than production |
| App Service (`Microsoft.Web/sites`, kind: app) | **Create new** | Needs its own URL and configuration |
| Function App (`Microsoft.Web/sites`, kind: functionapp) | **Create new** | Needs its own storage and config |
| SQL Server (`Microsoft.Sql/servers`) | **Reuse production server** | ~$150+/month saved; create new DBs on it |
| SQL Database | **Create new or copy from prod** | See Phase 4 for both paths |
| Application Insights | **Create new** | Keep staging telemetry separate |
| Storage Account | **Create new** | Required by Function App runtime |
| Communication Services | **Reuse from production** | Zero cost to reference cross-RG |
| Email Communication Services | **Reuse from production** | Zero cost to reference cross-RG |
| Data Factory | **Skip** (unless specifically required) | Expensive; rarely needed in staging |
| Key Vault | **Ask user** | New with separate secrets, or shared read access |
| Action groups / Alert rules | **Skip** | Not needed for non-production |

Confirm this plan with the user before running any commands.

## Phase 4: Create Azure Resources

### Resource Group
```bash
az group create --name <new-rg> --location <region>
```

### App Service Plan
```bash
az appservice plan create \
  --name <prefix>-plan \
  --resource-group <new-rg> \
  --location <region> \
  --sku B1 \
  --is-linux    # omit for Windows-hosted apps
```

### App Service (.NET)
```bash
az webapp create \
  --name <prefix>-app \
  --resource-group <new-rg> \
  --plan <prefix>-plan \
  --runtime "DOTNET|8"    # adjust to match target framework version
```

### Function App (if present in production)
```bash
az storage account create \
  --name <prefix>store \
  --resource-group <new-rg> \
  --location <region> \
  --sku Standard_LRS

az functionapp create \
  --name <prefix>-func \
  --resource-group <new-rg> \
  --storage-account <prefix>store \
  --consumption-plan-location <region> \
  --runtime dotnet \
  --runtime-version 8 \
  --functions-version 4
```

### SQL Databases

Ask the user which database strategy to use, then follow the matching path:

**Option A — Copy production data + scrub:**
```bash
# Copies land on the same server — no extra server cost
az sql db copy \
  --name <prod-db-name> \
  --resource-group <prod-rg> \
  --server <prod-sql-server> \
  --dest-name <prefix>-<db-name> \
  --dest-resource-group <prod-rg> \
  --dest-server <prod-sql-server>
```
After the copy completes, run environment-specific scrub scripts to anonymize or remove
sensitive production data before anyone uses the staging environment.

**Option B — Empty database + run migrations:**
```bash
az sql db create \
  --server <prod-sql-server-name> \
  --resource-group <prod-rg> \    # databases belong to the server's resource group
  --name <prefix>-<db-name> \
  --service-objective S0
```
EF Core migrations can run automatically on first deploy if the app is configured for it, or
manually after deploy: `dotnet ef database update --connection "<conn-string>"`

**Get connection string (both options):**
```bash
az sql db show-connection-string \
  --server <prod-sql-server-name> \
  --name <prefix>-<db-name> \
  --client ado.net
```

### Application Insights
```bash
az monitor app-insights component create \
  --app <prefix>-insights \
  --resource-group <new-rg> \
  --location <region>

az monitor app-insights component show \
  --app <prefix>-insights \
  --resource-group <new-rg> \
  --query connectionString -o tsv
```

### App Settings (Connection Strings + Config)
```bash
az webapp config appsettings set \
  --name <prefix>-app \
  --resource-group <new-rg> \
  --settings \
    "APPLICATIONINSIGHTS_CONNECTION_STRING=<conn-str>" \
    "ASPNETCORE_ENVIRONMENT=Staging"
    # Add database connection strings and any other required config keys here
```

### Service Principal for GitHub Actions
```bash
az ad sp create-for-rbac \
  --name <prefix>-deploy-sp \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/<new-rg> \
  --sdk-auth
```
Save the full JSON output — this becomes `AZURE_CREDENTIALS` in GitHub.

## Phase 5: GitHub Setup

### Create Branch
```bash
git checkout <base-branch> && git pull
git checkout -b <new-branch>
git push -u origin <new-branch>
```

### Detect Existing CI/CD

Before creating a workflow, check what already exists:
```bash
ls .github/workflows/
```

- **No workflows directory / empty** → Create `.github/workflows/deploy.yml` using `references/workflow-dotnet.yml` as the template
- **Workflow file(s) exist** → Open the existing deploy workflow, add `<new-branch>` to the `on.push.branches` list; a matching GitHub Environment (created below) will supply the correct secrets automatically

### Create GitHub Environment + Secrets

**Critical:** Use GitHub Environments — not repo-level secrets. When one repo deploys to multiple
branches from the same workflow, repo-level secrets are shared across all branches, meaning every
push deploys to the same Azure site regardless of branch. GitHub Environments give each branch its
own isolated secret set. The branch name **must exactly match** the GitHub Environment name.

```bash
# Create the environment
gh api repos/<owner>/<repo>/environments/<new-branch> --method PUT

# Secrets scoped to this environment only
gh secret set AZURE_CREDENTIALS      --env <new-branch> --body '<service-principal-json>'
gh secret set AZURE_WEBAPP_NAME      --env <new-branch> --body '<prefix>-app'
gh secret set AZURE_RESOURCE_GROUP   --env <new-branch> --body '<new-rg>'
gh secret set AZURE_FUNCTIONAPP_NAME --env <new-branch> --body '<prefix>-func'
# Add any app-specific secrets (API keys, third-party credentials, etc.)
```

In the workflow, `environment: ${{ github.ref_name }}` auto-selects the matching environment and
loads its secrets based on the branch being pushed — no changes to the workflow file needed when
adding future environments.

### Workflow File

See `references/workflow-dotnet.yml` for the complete template. Key points:
- Run `dotnet publish` before deploying — use `package: ./publish`, not `package: .`
- `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true` suppresses Node.js 20 deprecation warnings in logs

## Phase 6: Verify

```bash
# Stream startup logs to catch config errors immediately
az webapp log tail --name <prefix>-app --resource-group <new-rg>
```

1. Push a commit to the new branch — watch the GitHub Actions run complete
2. Open the App Service URL and confirm the app loads
3. Confirm Application Insights receives telemetry
4. Verify database connectivity through normal app behavior

## Common Pitfalls

| Mistake | Symptom | Fix |
|---------|---------|-----|
| `package: .` instead of `package: ./publish` | App runs without compilation output | Always `dotnet publish` first; deploy the publish folder |
| `scm-do-build-during-deployment` on `azure/webapps-deploy@v3` | `Unexpected input(s)` workflow error | That param is `Azure/functions-action` only — remove it |
| `az webapp show --name X` without `--resource-group` | `ERROR: --resource-group required` | Always pass `--resource-group`; az CLI cannot auto-discover it |
| Creating a new SQL server for staging | ~$150+/month wasted | Create new databases on the existing production server instead |
| Repo-level GitHub secrets instead of Environments | All branches deploy to the same Azure site | Use GitHub Environments with per-branch secrets |
| Branch name ≠ GitHub Environment name | Wrong secrets loaded; deploy hits wrong site | They must match exactly |
| Missing `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24` | Node.js 20 deprecation warnings in every run | Add as top-level `env:` in the workflow file |
| Skipping production RG analysis | Missing resources discovered after environment is live | Always run `az resource list` on production first |

## Cost Estimate

Typical per-environment cost (adjust for your region):

| Resource | SKU | ~Monthly USD |
|----------|-----|-------------|
| App Service Plan | B1 | ~$13 |
| SQL Database | S0 per database | ~$15 each |
| Function App | Consumption | ~free |
| Storage Account | LRS | <$1 |
| Application Insights | Free tier | ~free |
| **Total** | | **~$28–43+ depending on database count** |
