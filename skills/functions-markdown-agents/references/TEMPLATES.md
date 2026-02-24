# Templates and Reference Files

## azure.yaml

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/Azure/azure-dev/main/schemas/v1.0/azure.yaml.json

name: <project-name>
version: 0.1.0
hooks:
  prepackage:
    shell: sh
    run: ./infra/hooks/prepackage.sh
  postpackage:
    shell: sh
    run: rm -rf ./infra/tmp
services:
  api:
    project: ./infra/tmp
    language: python
    host: function
```

## .gitignore

```
# Python
__pycache__/
*.py[cod]
*$py.class
.venv/
venv/
ENV/
env/
.env/

# Azure Functions
local.settings.json
__azurite_db*__.json
__blobstorage__/
__queuestorage__/
__tablestorage__/
azurite_db_blob*.json
azurite_db_queue*.json
azurite_db_table*.json
*.debug.log
.python_packages/
bin/
obj/

# Node.js
node_modules/

# Secrets
*.pem
*.key
*.env
.env.*
secrets.json
credentials.json

# IDE / OS
.idea/
*.swp
*.swo
.DS_Store
Thumbs.db

# Test
.pytest_cache/
.coverage
htmlcov/

# Build
dist/
build/
*.egg-info/

# Azure / azd
.azure/
tmp/

# Runtime assets (specific files managed separately)
infra/assets/.gitignore
infra/assets/.vscode/
infra/assets/requirements.txt

# Compiled ARM template (regenerated from Bicep)
infra/main.json
```

## Key Files from the Sample Repo

When bootstrapping the `infra/` directory, copy these files from [Azure-Samples/functions-markdown-agent](https://github.com/Azure-Samples/functions-markdown-agent):

| File | Purpose |
|------|---------|
| `infra/main.bicep` | Main Bicep template: resource group, storage, function app, identity, monitoring, foundry |
| `infra/main.parameters.json` | Parameter bindings from azd environment variables |
| `infra/abbreviations.json` | Azure resource naming abbreviation prefixes |
| `infra/app/api.bicep` | Flex Consumption function app with managed identity and session file share |
| `infra/app/rbac.bicep` | Storage, queue, table, and App Insights role assignments |
| `infra/app/foundry.bicep` | Microsoft AI Foundry account and model deployment |
| `infra/app/vnet.bicep` | Optional VNet integration |
| `infra/app/storage-PrivateEndpoint.bicep` | Storage private endpoints for VNet |
| `infra/hooks/prepackage.sh` | Copies agent files → infra/tmp and merges infra/assets (replace for root layout) |

Do **not** copy `infra/main.json` (compiled ARM template — it gets regenerated from Bicep).

`infra/assets/` IS required and must be copied from the sample repo. These are runtime files maintained separately from the user's agent code — do not modify them.
