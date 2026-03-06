# Connect project to Azure resources

Replace all placeholder values in a scaffolded project with real Azure resource IDs from a completed `project-config.json`.

## What this command does

Reads a completed `project-config.json` and replaces all `TODO_*` placeholder values throughout the codebase with real values. This is the final step that turns a buildable-but-disconnected scaffold into a fully wired-up project.

## Steps

1. **Find and read `project-config.json`:**
   - Look in the current directory
   - If not found, ask for the path
   - Validate that all required values are filled in (not placeholder GUIDs or `TODO_*` strings)
   - If any values are missing, list them and suggest running `/azure-setup` first

2. **Scan for placeholders:**
   - Search all files in the project for `TODO_*` placeholder strings
   - Build a replacement map from config values to placeholders

3. **Replace placeholders in these files:**

   **API (.NET):**
   - `appsettings.json` — Key Vault URI, Entra config, storage URIs, Cosmos config
   - `appsettings.Development.json` — local dev values (tenant, client IDs, storage URIs, admin group)
   - `Program.cs` — Key Vault name (if hardcoded)

   **Portal (React):**
   - `.env.development` — local API URL, tenant info, client IDs
   - `.env.production` — production API URL (APIM gateway or direct), tenant info, client IDs
   - `.env.example` — placeholder documentation

   **Pipeline:**
   - `azure-pipelines.yml` — resource group, app service names, agent pool

   **Config:**
   - `project-config.json` — replace any remaining TODOs

4. **Set up Key Vault secrets** (with confirmation):
   - Run `az keyvault secret set` for each secret defined in the config
   - Secrets: TenantSubdomain, TenantId, ClientId, AdminGroupId, Graph--ClientSecret

5. **Configure CORS:**
   - Run `az webapp cors add` for the API App Service with portal URL + localhost

6. **Verify the build:**
   - Run `dotnet build` — should still compile
   - Run `npm run build` in the portal directory — should still compile

7. **Summary:**
   - List all files modified and values replaced
   - List any manual steps still needed (e.g., "Push to master to trigger first pipeline run")
   - Remind about post-deploy steps (managed identity RBAC, admin consent, etc.)

## Placeholder → Config mapping

| Placeholder | Config path |
|-------------|-------------|
| `TODO_TENANT_SUBDOMAIN` | `entra.tenantSubdomain` |
| `TODO_TENANT_ID` | `entra.tenantId` |
| `TODO_API_CLIENT_ID` | `entra.apiAppRegistration.clientId` |
| `TODO_PORTAL_CLIENT_ID` | `entra.portalAppRegistration.clientId` |
| `TODO_ADMIN_GROUP_ID` | `entra.adminGroup.objectId` |
| `TODO_API_URL` | `appServices.api.url` |
| `TODO_PORTAL_URL` | `appServices.portal.url` |
| `TODO_KEYVAULT_URI` | `keyVault.uri` |
| `TODO_STORAGE_ACCOUNT` | `storage.accountName` |
| `TODO_COSMOS_URI` | `cosmosDb.accountUri` |
| `TODO_COSMOS_DB_NAME` | `cosmosDb.databaseName` |
| `TODO_APIM_GATEWAY_URL` | `apim.gatewayUrl` |

## Important notes

- Always show a summary of changes before writing files — let the user confirm
- Create a git commit after replacements (if in a git repo) so changes are easily reversible
- If some placeholders aren't found in any files, warn but don't fail — the scaffold may have evolved
