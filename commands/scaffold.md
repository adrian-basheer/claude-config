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

## Output

After completion, tell the user:
1. The project builds successfully with placeholder values
2. Next step: follow the `runsheets/` in order to create Azure resources
3. Then run `/azure-setup` for interactive guidance, or fill in `project-config.json` manually
4. Finally run `/azure-connect` to wire up the real values
