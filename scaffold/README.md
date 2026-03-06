# Scaffold Project

Generates a complete **ASP.NET Core 8.0 Web API + React SPA** project with:

- Microsoft Entra External ID (CIAM) authentication
- User approval workflow (admin approves new sign-ups)
- Role-based access control (Admin/Editor/Viewer, extensible)
- User management admin UI
- Azure Table Storage for user profiles
- Azure Key Vault for secrets (production)
- CI/CD pipeline (Azure DevOps)
- All known gotchas and lessons learned baked in from day one

## Quick Start

### 1. Set Up Azure Resources

Follow the runsheets in `runsheets/` in order. Fill in your values as you go.

### 2. Create Your Config File

Copy `project-config.template.json` to your project parent folder and fill in the values from the runsheets:

```bash
cp scaffold/project-config.template.json ~/source/repos/project-config.json
# Edit with your values
```

### 3. Generate the Project

*(Slash command coming soon)*

```
/scaffold-project MyApp ~/source/repos/project-config.json
```

This will create `~/source/repos/MyApp/` with the full project structure.

## Files

| File | Purpose |
|------|---------|
| `SCAFFOLD-SPEC.md` | Full specification — architecture patterns, design decisions, code patterns extracted from Mixshare |
| `project-config.template.json` | Template config file with placeholder values |
| `runsheets/00-overview.md` | Runsheet index with dependency graph and time estimates |
| `runsheets/01-entra-external-id-setup.md` | Create Entra External ID tenant |
| `runsheets/02-app-registrations.md` | Register API + Portal apps in Entra |
| `runsheets/03-storage-account.md` | Create Azure Storage Account with RBAC |
| `runsheets/04-key-vault.md` | Create Key Vault and store secrets |
| `runsheets/05-app-services.md` | Create API + Portal App Services |
| `runsheets/06-microsoft-graph-api.md` | Configure Graph API permissions |
| `runsheets/07-azure-devops-pipeline.md` | Set up CI/CD pipeline |
| `runsheets/08-api-management.md` | Set up APIM gateway (analytics, rate limiting) |

## What Gets Generated

See `SCAFFOLD-SPEC.md` for the complete output structure, code patterns, and architecture decisions.

### Not Included (By Design)

- **APIM included** — Consumption tier gateway for analytics/rate limiting; portal calls APIM in production, API directly in dev
- **No domain-specific business logic** — only the user management baseline
- **No ARM/Bicep templates** — manual resource creation via runsheets for now (planned for future)
- **No Docker** — deployed directly to Azure App Service
