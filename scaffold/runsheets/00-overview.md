# Azure Resource Setup — Runsheet Overview

Follow these runsheets **in order** to set up all Azure resources needed for your project. Each runsheet tells you which values to fill into your `project-config.json`.

## Order of Operations

```
1. Entra External ID Tenant    → tenantSubdomain, tenantId, adminGroupId
2. App Registrations            → apiClientId, portalClientId
3. Storage Account              → storage URIs
4. Key Vault                    → vault name/URI, store secrets
5. App Services                 → API + Portal URLs, managed identity + RBAC
6. Microsoft Graph API          → client secret for user management
7. Azure DevOps Pipeline        → service principal, variable group, agent
8. API Management (APIM)        → gateway URL, import API, update portal config
9. Cosmos DB                    → account URI, database, containers, managed identity
```

## Dependency Graph

```
[1. Entra Tenant]
    └──► [2. App Registrations]
             └──► [6. Graph API] (needs API app registration)

[3. Storage Account] ──────┐
                           ├──► [4. Key Vault] (stores Storage URIs + Entra secrets)
[1. Entra Tenant] ─────────┘

[4. Key Vault] ────────────┐
[3. Storage Account] ──────┤
                           └──► [5. App Services] (managed identity needs RBAC on both)

[4. Key Vault] ────────────┐
                           └──► [9. Cosmos DB] (stores URI + key; managed identity after step 5)
[5. App Services] ─────────┘

[All above] ───────────────► [7. Pipeline] (needs all resource names + service principal)

[5. App Services] ─────────► [8. APIM] (imports API from App Service, portal points here)
[7. Pipeline] ─────────────► [8. APIM] (update VITE_API_BASE_URL to APIM gateway)
```

## Time Estimate

| Step | Approx. Time |
|------|-------------|
| 1. Entra Tenant | 10-15 min |
| 2. App Registrations | 15-20 min |
| 3. Storage Account | 5-10 min |
| 4. Key Vault | 10 min |
| 5. App Services | 10-15 min |
| 6. Graph API | 10 min |
| 7. Pipeline | 15-20 min |
| 8. API Management | 10-15 min |
| 9. Cosmos DB | 5-10 min |
| **Total** | **~95-120 min** |

## After All Runsheets

Once all resources are set up and the pipeline is running:

1. Push your code to the trigger branch
2. Pipeline builds and deploys automatically
3. Navigate to the portal URL
4. Sign in — you should be auto-approved as an admin
5. Visit `/admin/users` to manage other users
6. Check APIM Analytics in Azure Portal for API traffic

## Request Flow (Production)

```
Browser → Portal (React SPA on App Service)
    │
    │  MSAL acquires JWT from Entra External ID
    │
    ▼
API Management (gateway)          ← analytics, rate limiting, policies
    │
    │  Passes Authorization header through
    │
    ▼
API App Service (ASP.NET Core)    ← JWT validation, approval middleware, permissions
    │
    ▼
Azure Table Storage / Graph API
```

In local development, the portal calls the API directly (no APIM).

## Troubleshooting Quick Reference

| Symptom | Likely Cause | Runsheet |
|---------|-------------|----------|
| CORS errors in browser | Missing CORS on APIM (prod) or API (dev) | 5, 8 |
| API 404 on all endpoints | Wrong zip structure in deployment | 5, 7 |
| "Insufficient privileges" in Graph | Missing admin consent | 6 |
| API 500 on startup | Missing Key Vault access or storage/Cosmos config | 3, 4, 9 |
| Cosmos DB 403 Forbidden | Missing data-plane RBAC role on managed identity | 9 |
| Cosmos DB 404 on startup | Database or container not yet created | 9 |
| Portal blank page | Missing web.config in deployment | 5, 7 |
| Admin not detected locally | Missing AdminGroupId in dev config | 1, 4 |
| Login loop / auth errors | Wrong redirect URIs or app registration config | 2 |
| 404 through APIM but works directly | Wrong API URL suffix in APIM | 8 |
| "Subscription key required" | Uncheck subscription requirement in APIM | 8 |
