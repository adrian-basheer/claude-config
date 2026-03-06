# Runsheet 4: Azure Key Vault

## Purpose

Create a Key Vault to securely store secrets (Entra config, Graph API client secret, Storage URIs). The API reads these at startup in production via managed identity.

## Prerequisites

- Azure subscription
- Resource group created

## Steps

### 1. Create the Key Vault

```bash
az keyvault create \
  --name {{keyVault.name}} \
  --resource-group {{azure.resourceGroup}} \
  --location {{azure.location}} \
  --enable-rbac-authorization true
```

Record the vault URI → `keyVault.uri`

### 2. Grant Yourself Access

```bash
MY_OID=$(az ad signed-in-user show --query id -o tsv)

az role assignment create \
  --assignee $MY_OID \
  --role "Key Vault Secrets Officer" \
  --scope $(az keyvault show --name {{keyVault.name}} --query id -o tsv)
```

### 3. Store Entra Configuration Secrets

```bash
az keyvault secret set --vault-name {{keyVault.name}} --name "Entra--TenantSubdomain" --value "{{entra.tenantSubdomain}}"
az keyvault secret set --vault-name {{keyVault.name}} --name "Entra--TenantId" --value "{{entra.tenantId}}"
az keyvault secret set --vault-name {{keyVault.name}} --name "Entra--ClientId" --value "{{entra.apiAppRegistration.clientId}}"
az keyvault secret set --vault-name {{keyVault.name}} --name "Entra--AdminGroupId" --value "{{entra.adminGroup.objectId}}"
```

### 4. Store Graph API Secret

After creating the client secret in Runsheet 6:

```bash
az keyvault secret set --vault-name {{keyVault.name}} --name "Graph--ClientSecret" --value "<client-secret-value>"
```

### 5. Store Storage URIs

See Runsheet 3, Step 5.

### 6. Grant API Managed Identity Access (After App Service Created)

After creating the API App Service (Runsheet 5):

```bash
API_PRINCIPAL_ID=$(az webapp identity show --name {{appServices.api.name}} --resource-group {{azure.resourceGroup}} --query principalId -o tsv)

KV_ID=$(az keyvault show --name {{keyVault.name}} --query id -o tsv)

az role assignment create \
  --assignee $API_PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope $KV_ID
```

## Key Vault Secret Naming Convention

Azure Key Vault uses `--` as the hierarchy delimiter (maps to `:` in .NET configuration):

| Secret Name | Maps to Config Key |
|-------------|-------------------|
| `Entra--TenantId` | `Entra:TenantId` |
| `Entra--ClientId` | `Entra:ClientId` |
| `Entra--TenantSubdomain` | `Entra:TenantSubdomain` |
| `Entra--AdminGroupId` | `Entra:AdminGroupId` |
| `Graph--ClientSecret` | `Graph:ClientSecret` |
| `StorageConnection--tableServiceUri` | `StorageConnection:tableServiceUri` |
| `StorageConnection--blobServiceUri` | `StorageConnection:blobServiceUri` |
| `StorageConnection--queueServiceUri` | `StorageConnection:queueServiceUri` |

## Verification

- [ ] Key Vault created with RBAC authorization enabled
- [ ] Your identity has Key Vault Secrets Officer role
- [ ] All Entra secrets stored
- [ ] Storage URIs stored
- [ ] API managed identity has Key Vault Secrets User role (after App Service)

## Common Issues

- **`Forbidden` on secret operations**: You need `Key Vault Secrets Officer` (not just `User`) to set secrets. `User` is read-only.
- **API can't read secrets at startup**: Check that the managed identity has `Key Vault Secrets User` role on the vault.
- **Secrets not appearing in config**: Key Vault is only loaded in non-Development (`!builder.Environment.IsDevelopment()`). For local dev, use `appsettings.Development.json` or user-secrets.

## Output → `project-config.json`

```json
{
  "keyVault": {
    "name": "{{keyVault.name}}",
    "uri": "https://{{keyVault.name}}.vault.azure.net/"
  }
}
```
