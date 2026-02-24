# Accessing Azure Resources from Tools

When Python tools need to access Azure resources (Cosmos DB, Storage, Key Vault, etc.), use managed identity with `DefaultAzureCredential`. Never hardcode credentials.

## Python Tool Pattern

```python
import os
from azure.identity.aio import DefaultAzureCredential
from azure.cosmos.aio import CosmosClient
from pydantic import BaseModel, Field


class QueryParams(BaseModel):
    query: str = Field(description="The query to run")


async def query_database(params: QueryParams) -> str:
    """Query the database for information."""
    credential = DefaultAzureCredential(
        managed_identity_client_id=os.environ.get("AZURE_CLIENT_ID")
    )
    client = CosmosClient(
        url=os.environ["COSMOS_ENDPOINT"],
        credential=credential
    )
    # ... use the client
    return "results"
```

Add the Azure SDK packages to `requirements.txt`:

```
pydantic>=2.0.0
azure-identity
azure-cosmos
```

## Bicep Changes

Three things need to happen in the infrastructure:

1. **Deploy the Azure resource** (e.g., Cosmos DB account) in `infra/main.bicep`
2. **Add role assignments** in `infra/app/rbac.bicep` for the function app's managed identity
3. **Pass the resource endpoint** as an app setting in `infra/main.bicep`

The managed identity on the Function App is already enabled in the base template.

### Example: Cosmos DB Role Assignment

Add to `infra/app/rbac.bicep`:

```bicep
// Add parameter
param cosmosAccountName string = ''

// Add role assignment
var cosmosDataContributorId = '00000000-0000-0000-0000-000000000004' // Cosmos DB Built-in Data Contributor

resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2024-05-15' existing = if (!empty(cosmosAccountName)) {
  name: cosmosAccountName
}

resource cosmosRoleAssignment 'Microsoft.DocumentDB/databaseAccounts/sqlRoleAssignments@2024-05-15' = if (!empty(cosmosAccountName)) {
  parent: cosmosAccount
  name: guid(cosmosAccount.id, managedIdentityPrincipalId, cosmosDataContributorId)
  properties: {
    roleDefinitionId: '${cosmosAccount.id}/sqlRoleDefinitions/${cosmosDataContributorId}'
    principalId: managedIdentityPrincipalId
    scope: cosmosAccount.id
  }
}
```

### Example: Pass Endpoint as App Setting

In `infra/main.bicep`, add to the `appSettings` union in the `api` module:

```bicep
appSettings: union(
  {
    // ... existing settings ...
    COSMOS_ENDPOINT: cosmosAccount.properties.documentEndpoint
  },
  // ... rest of union ...
)
```

## Common Azure Role Definition IDs

| Service | Role | ID |
|---------|------|----|
| Storage Blob | Data Owner | `b7e6dc6d-f1e8-4753-8033-0f276bb0955b` |
| Storage Queue | Data Contributor | `974c5e8b-45b9-4653-ba55-5f855dd0fb88` |
| Storage Table | Data Contributor | `0a9a7e1f-b9d0-4cc4-a60d-0319b160aaa3` |
| Cosmos DB | Data Contributor (built-in) | `00000000-0000-0000-0000-000000000004` |
| Key Vault | Secrets User | `4633458b-17de-408a-b874-0445c86b69e6` |
| App Insights | Monitoring Metrics Publisher | `3913510d-42f4-4e42-8a64-420c390055eb` |
