# Runsheet 6: Microsoft Graph API Setup

## Purpose

Configure the API to query users and group membership from the Entra External ID directory using Microsoft Graph. This enables the admin Users page and admin group detection.

## Prerequisites

- Entra External ID tenant with API app registration (Runsheets 1 & 2)
- Key Vault created (Runsheet 4)
- You must be in the **CIAM tenant directory**

## Steps

### 1. Add Application Permissions to API App Registration

1. Switch to the CIAM tenant directory
2. **Microsoft Entra ID** → **App registrations** → select `{{projectName}}Api`
3. Go to **API permissions** → **+ Add a permission**
4. Select **Microsoft Graph** → **Application permissions**
5. Add these permissions:
   - `User.Read.All` — read all user profiles
   - `GroupMember.Read.All` — read group membership (needed for admin detection)
6. Click **Add permissions**

### 2. Grant Admin Consent

1. Still on the **API permissions** page
2. Click **Grant admin consent for {{tenantName}}**
3. Confirm by clicking **Yes**

> **Important**: Adding a permission is just a *request*. You must also click "Grant admin consent" for it to take effect. Without consent, Graph API calls will fail with "Insufficient privileges".

### 3. Create a Client Secret

1. Go to **Certificates & secrets** → **+ New client secret**
2. Fill in:
   - **Description**: `Graph API access`
   - **Expires**: Choose appropriate duration (e.g. 24 months)
3. Click **Add**
4. **Immediately copy the secret value** — it won't be shown again

### 4. Store the Secret in Key Vault

```bash
az keyvault secret set \
  --vault-name {{keyVault.name}} \
  --name "Graph--ClientSecret" \
  --value "<the-secret-value-you-just-copied>"
```

### 5. Configure Local Development (User Secrets)

```bash
cd {{projectName}}Api
dotnet user-secrets set "Graph:ClientSecret" "<the-secret-value>"
```

### 6. Verify the Configuration

The API reads these config keys:

| Config Key | Source (Production) | Source (Dev) |
|------------|-------------------|-------------|
| `Graph:ClientId` | Falls back to `Entra:ClientId` | Falls back to `Entra:ClientId` |
| `Graph:ClientSecret` | Key Vault (`Graph--ClientSecret`) | User secrets |
| `Entra:TenantId` | Key Vault | appsettings.Development.json |

The `GraphService` creates a `ClientSecretCredential` with these values to authenticate with Graph API.

## How It Works in the API

```csharp
// GraphService constructor
var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
_graphClient = new GraphServiceClient(credential, new[] { "https://graph.microsoft.com/.default" });
```

The service provides:
- `GetUsersAsync()` — lists all users in the directory
- `GetUserAsync(id)` — gets a specific user
- `GetGroupMemberIdsAsync(groupId)` — gets member IDs of the admin group

## Verification

- [ ] `User.Read.All` permission added to API app registration
- [ ] `GroupMember.Read.All` permission added to API app registration
- [ ] Admin consent granted for both permissions
- [ ] Client secret created and value copied
- [ ] Secret stored in Key Vault as `Graph--ClientSecret`
- [ ] User secrets configured for local development
- [ ] API can list users via `GET /api/Users` (test with Swagger)

## Common Issues

- **"Insufficient privileges to complete the operation"**: Either admin consent wasn't granted, or you're missing `GroupMember.Read.All` for group membership queries.
- **Secret value not available**: You can only see the value immediately after creation. If you lost it, delete and create a new one.
- **API works locally but not in production**: Check that the Key Vault secret name uses `--` delimiter: `Graph--ClientSecret` (not `Graph:ClientSecret`).
- **Wrong tenant**: Make sure you're adding permissions in the CIAM tenant, not your main Azure AD tenant.
