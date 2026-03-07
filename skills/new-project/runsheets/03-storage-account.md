# Runsheet 3: Azure Storage Account

## Purpose

Create an Azure Storage Account for Table Storage (user profiles, permissions) and optionally Blob/Queue/File storage. Uses managed identity (RBAC) — no connection strings.

## Prerequisites

- Azure subscription
- Resource group created

## Steps

### 1. Create the Storage Account

```bash
az storage account create \
  --name {{storage.accountName}} \
  --resource-group {{azure.resourceGroup}} \
  --location {{azure.location}} \
  --sku Standard_LRS \
  --kind StorageV2
```

Record the storage account name → `storage.accountName`

### 2. Note the Service URIs

The URIs follow a standard pattern:

| Service | URI | Config Key |
|---------|-----|------------|
| Table | `https://{{storage.accountName}}.table.core.windows.net/` | `storage.tableServiceUri` |
| Blob | `https://{{storage.accountName}}.blob.core.windows.net/` | `storage.blobServiceUri` |
| Queue | `https://{{storage.accountName}}.queue.core.windows.net/` | `storage.queueServiceUri` |

### 3. Grant Your Developer Identity Access (for Local Dev)

Your local `az login` identity needs RBAC roles to access storage via `DefaultAzureCredential`.

```bash
# Get your object ID
MY_OID=$(az ad signed-in-user show --query id -o tsv)

# Get the storage account resource ID
STORAGE_ID=$(az storage account show --name {{storage.accountName}} --resource-group {{azure.resourceGroup}} --query id -o tsv)

# Grant Table Data Contributor (required for Table Storage)
az role assignment create \
  --assignee $MY_OID \
  --role "Storage Table Data Contributor" \
  --scope $STORAGE_ID

# Grant Blob Data Contributor (if using Blob Storage)
az role assignment create \
  --assignee $MY_OID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID
```

### 4. Configure Local Development (User Secrets)

```bash
cd {{projectName}}Api

dotnet user-secrets set "StorageConnection:tableServiceUri" "https://{{storage.accountName}}.table.core.windows.net/"
dotnet user-secrets set "StorageConnection:blobServiceUri" "https://{{storage.accountName}}.blob.core.windows.net/"
dotnet user-secrets set "StorageConnection:queueServiceUri" "https://{{storage.accountName}}.queue.core.windows.net/"
```

### 5. Store URIs in Key Vault (for Production)

```bash
az keyvault secret set \
  --vault-name {{keyVault.name}} \
  --name "StorageConnection--tableServiceUri" \
  --value "https://{{storage.accountName}}.table.core.windows.net/"

az keyvault secret set \
  --vault-name {{keyVault.name}} \
  --name "StorageConnection--blobServiceUri" \
  --value "https://{{storage.accountName}}.blob.core.windows.net/"

az keyvault secret set \
  --vault-name {{keyVault.name}} \
  --name "StorageConnection--queueServiceUri" \
  --value "https://{{storage.accountName}}.queue.core.windows.net/"
```

### 6. Grant API Managed Identity Access (After App Service Created)

After creating the API App Service (Runsheet 5), grant its managed identity access to **each storage service it uses**. Each service (Table, Blob, Queue) requires its own RBAC role.

1. Go to Azure Portal → Storage Account → **Access Control (IAM)**
2. Click **+ Add** → **Add role assignment**
3. Add **each** role the API needs:

| Role | Required For |
|------|-------------|
| **Storage Table Data Contributor** | Table Storage (user profiles, permissions, reference data) |
| **Storage Blob Data Contributor** | Blob Storage (file uploads — case files, FSA, CSV) |
| **Storage Queue Data Contributor** | Queue Storage (if using queues for background jobs) |

4. Members: Select **Managed identity** → **App Service** → select `{{appServices.api.name}}`
5. Review + Assign
6. **Repeat for each role** — they must be assigned individually

Or via CLI:

```bash
API_PRINCIPAL_ID=$(az webapp identity show --name {{appServices.api.name}} --resource-group {{azure.resourceGroup}} --query principalId -o tsv)

# Table Storage access
az role assignment create \
  --assignee $API_PRINCIPAL_ID \
  --role "Storage Table Data Contributor" \
  --scope $STORAGE_ID

# Blob Storage access
az role assignment create \
  --assignee $API_PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID

# Queue Storage access (if needed)
az role assignment create \
  --assignee $API_PRINCIPAL_ID \
  --role "Storage Queue Data Contributor" \
  --scope $STORAGE_ID
```

> **Common mistake**: Granting only Table access and assuming it covers Blob/Queue. Each storage service has its own RBAC role — they are not inherited.

## Verification

- [ ] Storage account created
- [ ] Your dev identity has Table Data Contributor + Blob Data Contributor roles
- [ ] User secrets configured for local dev
- [ ] URIs stored in Key Vault (after Key Vault created)
- [ ] API managed identity has Table Data Contributor role (after App Service created)
- [ ] API managed identity has Blob Data Contributor role (after App Service created)
- [ ] API managed identity has Queue Data Contributor role if using queues (after App Service created)

## Common Issues

- **`SocketException: Connection refused (127.0.0.1:10002)`**: You're pointing to Azurite (local emulator) which isn't running. Use real Azure Storage URIs instead.
- **`AuthorizationPermissionMismatch`**: Your identity doesn't have the right RBAC role. Use `az role assignment create` as shown above.
- **`MissingSubscription` error on role assignment**: Use the object ID (`--assignee <oid>`) instead of UPN format.
- **`ArgumentNullException: connectionString`**: You're using the string overload instead of the URI overload. The scaffold code uses `new Uri(...)` overloads — don't change this.

## Output → `project-config.json`

```json
{
  "storage": {
    "accountName": "{{storage.accountName}}",
    "tableServiceUri": "https://{{storage.accountName}}.table.core.windows.net/",
    "blobServiceUri": "https://{{storage.accountName}}.blob.core.windows.net/",
    "queueServiceUri": "https://{{storage.accountName}}.queue.core.windows.net/"
  }
}
```
