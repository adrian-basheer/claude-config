# Runsheet 2: Entra App Registrations

## Purpose

Register two applications in Entra External ID — one for the API (validates tokens, exposes scopes) and one for the SPA portal (authenticates users, requests tokens).

## Prerequisites

- Entra External ID tenant created (Runsheet 1)
- You must be in the **CIAM tenant directory** (not your main Azure AD)

## Steps

### Part A: API App Registration

#### 1. Create the Registration

1. Switch to the CIAM tenant directory
2. **Microsoft Entra ID** → **App registrations** → **+ New registration**
3. Fill in:
   - **Name**: `{{projectName}}Api`
   - **Supported account types**: Accounts in this organizational directory only
4. Click **Register**
5. Note the **Application (client) ID** → `entra.apiAppRegistration.clientId`

#### 2. Expose an API Scope

1. Go to **Expose an API**
2. Click **+ Add** next to "Application ID URI" — accept the default `api://<client-id>`
3. Click **+ Add a scope**:
   - **Scope name**: `access_as_user`
   - **Who can consent**: Admins and users
   - **Admin consent display name**: Access the API
   - **Admin consent description**: Allows the app to access the API on behalf of the signed-in user
4. Click **Add scope**

#### 3. Configure Token Claims

1. Go to **Token configuration** → **+ Add groups claim**
2. Select **Security groups**
3. Under **Customize token properties by type**:
   - For **Access** token: check **Group ID**
4. Save

### Part B: Portal (SPA) App Registration

#### 1. Create the Registration

1. **App registrations** → **+ New registration**
2. Fill in:
   - **Name**: `{{projectName}}Portal`
   - **Supported account types**: Accounts in this organizational directory only
   - **Redirect URI**: Select **Single-page application (SPA)** and enter `http://localhost:5173`
3. Click **Register**
4. Note the **Application (client) ID** → `entra.portalAppRegistration.clientId`

#### 2. Add Production Redirect URI

1. Go to **Authentication**
2. Under **Single-page application**, click **+ Add URI**
3. Add: `https://{{appServices.portal.name}}.azurewebsites.net`
4. Save

#### 3. Grant API Permission

1. Go to **API permissions** → **+ Add a permission**
2. Select **My APIs** → select `{{projectName}}Api`
3. Check `access_as_user`
4. Click **Add permissions**
5. Click **Grant admin consent for {{tenantName}}**

### Part C: Link to User Flow

1. Go to **External Identities** → **User flows** → select `signupsignin1`
2. Under **Applications**, click **+ Add application**
3. Add both `{{projectName}}Api` and `{{projectName}}Portal`

## Verification

- [ ] API app registration created with client ID recorded
- [ ] API scope `access_as_user` exposed
- [ ] Groups claim configured on access token
- [ ] Portal app registration created with client ID recorded
- [ ] Redirect URIs include `localhost:5173` and production URL
- [ ] Portal has API permission with admin consent granted
- [ ] Both apps linked to user flow

## Output → `project-config.json`

```json
{
  "entra": {
    "apiAppRegistration": {
      "clientId": "<from Part A step 1>",
      "displayName": "{{projectName}}Api"
    },
    "portalAppRegistration": {
      "clientId": "<from Part B step 1>",
      "displayName": "{{projectName}}Portal"
    },
    "apiScope": "access_as_user"
  }
}
```

## Common Issues

- **"Insufficient privileges"**: Make sure you're in the CIAM tenant, not your main Azure AD
- **Scope not showing in "My APIs"**: You need to add the scope in the API registration first (Part A step 2)
- **Admin consent button greyed out**: You need Global Administrator or Privileged Role Administrator in the CIAM tenant
