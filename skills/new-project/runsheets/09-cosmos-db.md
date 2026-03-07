# Runsheet 9: Azure Cosmos DB

## Purpose

Create an Azure Cosmos DB account (NoSQL API, serverless) for case data storage. Cases are stored as JSON documents with a partition key of `/submittedBy` (the user's Entra Object ID). Uses managed identity in production — no connection strings.

## Prerequisites

- Azure subscription
- Resource group created
- Key Vault created (Runsheet 4) — for storing the account key
- API App Service created (Runsheet 5) — for managed identity RBAC

---

## Steps

### 1. Create the Cosmos DB Account

Serverless tier — pay per request unit, no idle cost. Suitable for low-to-medium traffic community platforms.

```bash
az cosmosdb create \
  --name {{cosmosDb.accountName}} \
  --resource-group {{azure.resourceGroup}} \
  --locations regionName={{azure.location}} failoverPriority=0 isZoneRedundant=false \
  --default-consistency-level Session \
  --kind GlobalDocumentDB \
  --capabilities EnableServerless
```

Record the account name → `cosmosDb.accountName`

### 2. Create the Database

```bash
az cosmosdb sql database create \
  --account-name {{cosmosDb.accountName}} \
  --resource-group {{azure.resourceGroup}} \
  --name {{cosmosDb.databaseName}}
```

### 3. Create the Cases Container

> **Windows / Git Bash users**: The leading `/` in the partition key path is misinterpreted as a file path by Git Bash. Prefix the command with `MSYS_NO_PATHCONV=1` to prevent this.

```bash
MSYS_NO_PATHCONV=1 az cosmosdb sql container create \
  --account-name {{cosmosDb.accountName}} \
  --resource-group {{azure.resourceGroup}} \
  --database-name {{cosmosDb.databaseName}} \
  --name {{cosmosDb.containers.cases}} \
  --partition-key-path "/submittedBy"
```

### 4. Note the Endpoint URI

The endpoint follows a standard pattern:

| Value | URI |
|-------|-----|
| Account URI | `https://{{cosmosDb.accountName}}.documents.azure.com:443/` |

Update `cosmosDb.accountUri` in your `project-config.json`.

### 5. Store URI and Key in Key Vault

```bash
# Store the account URI
az keyvault secret set \
  --vault-name {{keyVault.name}} \
  --name "CosmosDb--AccountUri" \
  --value "https://{{cosmosDb.accountName}}.documents.azure.com:443/"

# Retrieve the primary key
COSMOS_KEY=$(az cosmosdb keys list \
  --name {{cosmosDb.accountName}} \
  --resource-group {{azure.resourceGroup}} \
  --query primaryMasterKey -o tsv)

# Store the key (used for local dev; production uses managed identity)
az keyvault secret set \
  --vault-name {{keyVault.name}} \
  --name "CosmosDb--AccountKey" \
  --value "$COSMOS_KEY"
```

### 6. Configure Local Development

Add the URI and key to `appsettings.Development.json` (this file is gitignored):

```json
"CosmosDb": {
  "AccountUri": "https://{{cosmosDb.accountName}}.documents.azure.com:443/",
  "AccountKey": "<paste key from step 5>"
}
```

The `AccountKey` in `appsettings.json` (committed) should remain empty — production authenticates via managed identity and does not need it.

### 7. Grant API Managed Identity Access (Production)

After creating the API App Service (Runsheet 5), grant its managed identity the **Cosmos DB Built-in Data Contributor** role. This is a Cosmos DB data-plane role, not an ARM RBAC role — it requires a different command.

```bash
# Get the API App Service managed identity principal ID
API_PRINCIPAL_ID=$(az webapp identity show \
  --name {{appServices.api.name}} \
  --resource-group {{azure.resourceGroup}} \
  --query principalId -o tsv)

# Get the Cosmos DB account resource ID
COSMOS_ID=$(az cosmosdb show \
  --name {{cosmosDb.accountName}} \
  --resource-group {{azure.resourceGroup}} \
  --query id -o tsv)

# Assign Cosmos DB Built-in Data Contributor (role ID: ...000000000002)
MSYS_NO_PATHCONV=1 az cosmosdb sql role assignment create \
  --account-name {{cosmosDb.accountName}} \
  --resource-group {{azure.resourceGroup}} \
  --role-definition-id "00000000-0000-0000-0000-000000000002" \
  --principal-id "$API_PRINCIPAL_ID" \
  --scope "$COSMOS_ID"
```

> **Note**: Cosmos DB data-plane RBAC is separate from Azure RBAC. The built-in role IDs are fixed GUIDs:
> - `...000000000001` = Cosmos DB Built-in Data **Reader**
> - `...000000000002` = Cosmos DB Built-in Data **Contributor** (read + write)

---

## Verification

- [ ] Cosmos DB account created (`az cosmosdb show --name {{cosmosDb.accountName}} --resource-group {{azure.resourceGroup}}`)
- [ ] Database `{{cosmosDb.databaseName}}` created
- [ ] Container `{{cosmosDb.containers.cases}}` created with partition key `/submittedBy`
- [ ] URI and key stored in Key Vault
- [ ] `appsettings.Development.json` populated with URI and key
- [ ] API starts locally without errors (`dotnet run --project {{projectName}}Api`)
- [ ] `POST /api/cases` in Swagger returns 201 with a case document
- [ ] API managed identity has Data Contributor role (after App Service created)

---

## Common Issues

- **`MSYS_NO_PATHCONV` on Windows**: Git Bash expands `/submittedBy` to `C:/Program Files/Git/submittedBy`. Always prefix partition key commands with `MSYS_NO_PATHCONV=1`.
- **`CosmosException 403 Forbidden`**: The identity (managed identity or dev user) does not have the Cosmos DB data-plane role. Run step 7 (managed identity) or verify the account key is correct (local dev).
- **`CosmosException 404 Not Found` on startup**: The database or container doesn't exist. Run steps 2–3, or let `CaseService.EnsureContainerExistsAsync()` create them on first run.
- **`ArgumentException: AccountEndpoint is required`**: `CosmosDb:AccountUri` is empty in config. Ensure `appsettings.Development.json` is populated for local dev.
- **`DefaultAzureCredential` fails locally**: You're trying to use managed identity on a local machine. For local dev, always provide `AccountKey` in `appsettings.Development.json`.

---

## Output → `project-config.json`

```json
{
  "cosmosDb": {
    "accountName": "{{cosmosDb.accountName}}",
    "accountUri": "https://{{cosmosDb.accountName}}.documents.azure.com:443/",
    "databaseName": "{{cosmosDb.databaseName}}",
    "containers": {
      "cases": "cases"
    }
  }
}
```
