---
description: Scaffold a new ASP.NET Core + React project with Azure infrastructure and Entra External ID auth
disable-model-invocation: true
---

# New Project

Create a fully wired-up ASP.NET Core 8.0 Web API + React SPA project with Entra External ID authentication, user approval workflow, RBAC, and CI/CD pipeline.

This skill guides you through three phases:
1. **Scaffold** — generate the codebase with placeholder values
2. **Azure Setup** — create Azure resources interactively
3. **Azure Connect** — replace placeholders with real values

## Progress Tracking

Progress is persisted in the project's `project-config.json` via the `_progress` field:

```json
"_progress": {
  "phase": 2,
  "lastCompletedRunsheet": "03-storage-account",
  "status": "in-progress"
}
```

**On every invocation**, read `project-config.json` first and resume from where `_progress` indicates. Update `_progress` after each milestone:
- After Phase 1 completes: `{ "phase": 1, "lastCompletedRunsheet": null, "status": "complete" }`
- During Phase 2, after each runsheet: `{ "phase": 2, "lastCompletedRunsheet": "03-storage-account", "status": "in-progress" }`
- After Phase 2 completes: `{ "phase": 2, "lastCompletedRunsheet": "09-cosmos-db", "status": "complete" }`
- After Phase 3 completes: `{ "phase": 3, "lastCompletedRunsheet": "09-cosmos-db", "status": "complete" }`

If `project-config.json` doesn't exist yet, start from Phase 1. If it exists, tell the user where they left off and confirm before continuing.

---

## Phase 1: Scaffold

Generate a buildable codebase with placeholder values. No Azure resources need to exist yet.

### Steps

1. **Ask the user for:**
   - **Project name** (e.g. `MyApp`) — used for solution, projects, namespaces
   - **Target directory** — where to create the project folder (defaults to current directory)
   - **Optional customizations** — theme colors, role definitions, or accept defaults

2. **Read the scaffold spec and template:**
   - Full spec: `${CLAUDE_SKILL_DIR}/SCAFFOLD-SPEC.md`
   - Config template: `${CLAUDE_SKILL_DIR}/project-config.template.json`

3. **Generate the project** following the spec exactly:
   - Create the folder structure from the Output Structure section
   - Generate all .NET projects (Core, Application, Infra, Api, Api.Tests) with correct layer dependencies
   - Generate the React portal with all components, stores, pages, and config
   - Generate `azure-pipelines.yml` from the CI/CD section
   - Generate `CLAUDE.md` pre-filled with project-specific instructions
   - Generate `lessons-learned.md` seeded with all known gotchas
   - Use placeholder values for Azure IDs (see Placeholder Convention below)
   - Generate `project-config.json` from the template with the project name filled in

4. **Copy runsheets** from `${CLAUDE_SKILL_DIR}/runsheets/` into the project's `runsheets/` folder (renaming references to use the project name)

5. **Verify the build:**
   - Run `dotnet build` from the project root
   - Run `npm install && npm run build` from the portal directory
   - Fix any compilation errors before proceeding

6. **Update `_progress`** in `project-config.json` to `{ "phase": 1, "status": "complete" }`

7. **Ask the user** if they want to continue to Phase 2 now, or stop here and resume later.

### Placeholder Convention

Use these exact placeholder strings so Phase 3 can find-and-replace them:

| Placeholder | Replaced by |
|-------------|-------------|
| `TODO_TENANT_SUBDOMAIN` | Entra tenant subdomain |
| `TODO_TENANT_ID` | Entra tenant ID (GUID) |
| `TODO_API_CLIENT_ID` | API app registration client ID |
| `TODO_PORTAL_CLIENT_ID` | Portal app registration client ID |
| `TODO_ADMIN_GROUP_ID` | Admin security group object ID |
| `TODO_API_URL` | API App Service URL |
| `TODO_PORTAL_URL` | Portal App Service URL |
| `TODO_KEYVAULT_URI` | Key Vault URI |
| `TODO_STORAGE_ACCOUNT` | Storage account name |
| `TODO_COSMOS_URI` | Cosmos DB account URI |
| `TODO_COSMOS_DB_NAME` | Cosmos DB database name |
| `TODO_APIM_GATEWAY_URL` | API Management gateway URL |

### Default Roles

Generate three built-in roles:

| Role | Description | Permissions |
|------|-------------|-------------|
| **Super Admin** | Full access | `users:read`, `users:manage`, `roles:assign`, `content:*`, `settings:manage`, `admin:*` |
| **Contributor** | Create and manage content | `content:read`, `content:create`, `content:edit`, `content:delete` |
| **Reader** | View content | `content:read` |

- Default role on approval: **Reader**
- Entra admin group members: auto-approved with **Super Admin**

### Debug Personas (Development Only)

Generate a dev-only persona system (only active when `ASPNETCORE_ENVIRONMENT=Development`):

| Persona | Status | Role |
|---------|--------|------|
| Alice Admin | approved | Super Admin |
| Bob Builder | approved | Contributor |
| Carol Viewer | approved | Reader |
| Dave Newbie | pending | — |
| Eve Blocked | suspended | — |

**Files**: `DevPersonaMiddleware.cs`, `DevController.cs`, `DevPersonaSelector.tsx`, `appsettings.Development.json`

**Safety**: Middleware only registered in dev, controller hidden from Swagger, React component checks `import.meta.env.DEV`.

---

## Phase 2: Azure Setup

Walk through Azure resource creation step-by-step, filling in `project-config.json` with real values.

### Steps

1. **Read `project-config.json`** from the project directory. Check what's already filled in vs placeholder.

2. **Read the runsheets** from the project's `runsheets/` folder:
   - Start with `00-overview.md` to show the dependency graph
   - Determine which steps are already done

3. **For each runsheet, in order:**

   a. **Show a summary** of what this step creates

   b. **For manual steps** (Azure Portal actions):
      - Explain what to do with specific navigation paths
      - Wait for user confirmation
      - Ask them to paste back generated values (GUIDs, URLs, etc.)

   c. **For automatable steps** (`az` CLI):
      - Show the command and explain what it does
      - Ask for confirmation before running
      - Capture output values automatically

   d. **Update `project-config.json`** after each runsheet so progress isn't lost
      - Update `_progress.lastCompletedRunsheet` to the runsheet just finished
      - Show what was filled in

   e. **Validate** — check GUIDs are valid format, URLs are well-formed

4. **Track progress:** After each runsheet show "Completed 3/9 — next: Key Vault". The `_progress` field in `project-config.json` records exactly where you are. On resume (even in a new session), read `_progress` and skip completed steps.

5. **After all runsheets:** Display completed config and ask if user wants to continue to Phase 3.

### Important notes

- Never run destructive Azure commands without confirmation
- If a resource already exists, detect and skip
- If `az` is not logged in, help the user log in first
- Some steps require Azure Portal and cannot be automated
- The user may not complete all steps in one session

---

## Phase 3: Azure Connect

Replace all placeholder values with real Azure resource IDs from the completed `project-config.json`.

### Steps

1. **Read `project-config.json`:**
   - Validate all required values are filled in
   - If any are missing, list them and suggest going back to Phase 2

2. **Scan for `TODO_*` placeholders** across the project and build a replacement map.

3. **Replace placeholders in:**

   **API (.NET):**
   - `appsettings.json` — Key Vault URI, Entra config, storage, Cosmos
   - `appsettings.Development.json` — local dev values
   - `Program.cs` — Key Vault name if hardcoded

   **Portal (React):**
   - `.env.development` — local API URL, tenant info, client IDs
   - `.env.production` — production values
   - `.env.example` — placeholder documentation

   **Pipeline:**
   - `azure-pipelines.yml` — resource group, app service names, agent pool

4. **Set up Key Vault secrets** (with confirmation):
   - `az keyvault secret set` for TenantSubdomain, TenantId, ClientId, AdminGroupId, Graph--ClientSecret

5. **Configure CORS:**
   - `az webapp cors add` for the API App Service with portal URL + localhost

6. **Verify the build:**
   - `dotnet build`
   - `npm run build` in portal directory

7. **Summary:**
   - List all files modified and values replaced
   - List manual steps still needed (first pipeline run, managed identity RBAC, admin consent)
   - Create a git commit with the changes

### Placeholder → Config mapping

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

### Important notes

- Show a summary of changes before writing — let the user confirm
- Create a git commit after replacements so changes are reversible
- If some placeholders aren't found, warn but don't fail

---

## Output

After Phase 1, tell the user:
1. The project builds successfully with placeholder values
2. Next: continue to Phase 2, or follow `runsheets/` manually

After Phase 3, tell the user:
1. All placeholders replaced with real values
2. Project builds with real config
3. Remaining manual steps (push to trigger pipeline, managed identity roles, admin consent)
