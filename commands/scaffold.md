# Scaffold a new project

Generate a new ASP.NET Core 8.0 Web API + React SPA project with Entra External ID authentication, user approval workflow, RBAC, and CI/CD pipeline.

## What this command does

Creates a fully buildable codebase with **placeholder values** for all Azure resource IDs. No Azure resources need to exist yet. The generated project includes a `runsheets/` folder with step-by-step guides for Azure setup.

## Steps

1. **Ask the user for:**
   - **Project name** (e.g. `MyApp`) — used to name the solution, projects, namespaces, and replace all template references
   - **Target directory** — where to create the project folder (defaults to current directory)
   - **Optional customizations** — theme colors, role definitions, or accept defaults from the template

2. **Read the scaffold spec:**
   - Full spec: `C:\Users\adria\source\repos\claude-config\scaffold\SCAFFOLD-SPEC.md`
   - Config template: `C:\Users\adria\source\repos\claude-config\scaffold\project-config.template.json`

3. **Generate the project** following the spec exactly:
   - Create the folder structure from the Output Structure section
   - Generate all .NET projects (Core, Application, Infra, Api, Api.Tests) with correct layer dependencies
   - Generate the React portal with all components, stores, pages, and config
   - Generate `azure-pipelines.yml` from the CI/CD section
   - Generate `CLAUDE.md` pre-filled with project-specific instructions
   - Generate `lessons-learned.md` seeded with all known gotchas
   - Use placeholder values for Azure IDs: `TODO_TENANT_ID`, `TODO_CLIENT_ID`, `TODO_API_URL`, etc.
   - Generate a `project-config.json` from the template with the project name filled in but Azure values as placeholders
   - Generate default roles and debug personas (see below)

4. **Copy runsheets** from `C:\Users\adria\source\repos\claude-config\scaffold\runsheets\` into the project's `runsheets/` folder (renaming references to use the project name)

5. **Verify the build:**
   - Run `dotnet build` from the project root
   - Run `npm install && npm run build` from the portal directory
   - Fix any compilation errors before considering done

## Placeholder convention

Use these exact placeholder strings so `/azure-connect` can find-and-replace them:

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

## Default Roles

The scaffold generates three built-in roles in the `UserProfileService.RoleDefinitions` dictionary and in `project-config.json`. These are seeded from the start so the app has a working RBAC system out of the box.

| Role | Description | Permissions |
|------|-------------|-------------|
| **Super Admin** | Full access to everything — user management, system settings, all content operations | `users:read`, `users:manage`, `roles:assign`, `content:read`, `content:create`, `content:edit`, `content:delete`, `settings:manage`, `admin:users`, `admin:settings` |
| **Contributor** | Create, edit, and manage content | `content:read`, `content:create`, `content:edit`, `content:delete` |
| **Reader** | Browse and view content in read-only mode | `content:read` |

- Users approved via the admin panel receive the **Reader** role by default (configurable in `project-config.json` → `defaultRoleOnApproval`)
- Entra admin group members are auto-approved with **Super Admin** role
- The portal's `PermissionGate` and `ProtectedRoute` components enforce these permissions in the UI
- The API's `UserApprovalMiddleware` enforces approval status; controllers can check roles/permissions for finer control

## Debug Personas (Development Only)

The scaffold generates a dev-only persona system so developers can test the app as different user types without needing multiple Entra accounts. This is **only active when `ASPNETCORE_ENVIRONMENT=Development`**.

### How it works

**API side** — `DevPersonaMiddleware` (registered before `UserApprovalMiddleware` in dev only):
- Checks for an `X-Dev-Persona` header on incoming requests
- If present, overrides the authenticated user's claims with the persona's claims
- Personas are defined in `appsettings.Development.json`
- In production, this middleware is not registered — the header is ignored

**Portal side** — `DevPersonaSelector` component (shown only in development):
- A floating dropdown in the bottom-right corner (only rendered when `import.meta.env.DEV` is true)
- Lists available personas fetched from `GET /api/dev/personas`
- Selecting a persona sets the `X-Dev-Persona` header on all subsequent API calls via `fetchClient`
- "Self" option clears the override and uses the real logged-in user

### Seed Personas

Generate these in `appsettings.Development.json`:

```json
{
  "DevPersonas": [
    {
      "id": "persona-superadmin",
      "name": "Alice Admin",
      "email": "alice@example.com",
      "status": "approved",
      "roles": ["Super Admin"]
    },
    {
      "id": "persona-contributor",
      "name": "Bob Builder",
      "email": "bob@example.com",
      "status": "approved",
      "roles": ["Contributor"]
    },
    {
      "id": "persona-reader",
      "name": "Carol Viewer",
      "email": "carol@example.com",
      "status": "approved",
      "roles": ["Reader"]
    },
    {
      "id": "persona-pending",
      "name": "Dave Newbie",
      "email": "dave@example.com",
      "status": "pending",
      "roles": []
    },
    {
      "id": "persona-suspended",
      "name": "Eve Blocked",
      "email": "eve@example.com",
      "status": "suspended",
      "roles": []
    }
  ]
}
```

### Files to generate

| File | Layer | Purpose |
|------|-------|---------|
| `Middleware/DevPersonaMiddleware.cs` | Api | Reads `X-Dev-Persona` header, overrides user claims |
| `Controllers/DevController.cs` | Api | `GET /api/dev/personas` — returns persona list (dev only) |
| `components/dev/DevPersonaSelector.tsx` | Portal | Floating dropdown to switch personas |
| `appsettings.Development.json` | Api | Persona definitions |

### Safety

- `DevPersonaMiddleware` is only registered in `Program.cs` inside an `if (app.Environment.IsDevelopment())` block
- `DevController` has `[ApiExplorerSettings(IgnoreApi = true)]` so it doesn't appear in Swagger/OpenAPI
- The portal component checks `import.meta.env.DEV` and renders nothing in production builds
- No persona code is included in release builds or production deployments

## Output

After completion, tell the user:
1. The project builds successfully with placeholder values
2. Next step: follow the `runsheets/` in order to create Azure resources
3. Then run `/azure-setup` for interactive guidance, or fill in `project-config.json` manually
4. Finally run `/azure-connect` to wire up the real values
