# Runsheet 5: Azure App Services

## Purpose

Create two Azure App Services — one for the ASP.NET Core API and one for the React SPA portal.

## Prerequisites

- Azure subscription
- Resource group created

## Steps

### Part A: API App Service

#### 1. Create the App Service Plan + Web App

```bash
# Create App Service Plan (free tier for dev/test)
az appservice plan create \
  --name {{appServices.api.name}}-plan \
  --resource-group {{azure.resourceGroup}} \
  --location {{azure.location}} \
  --sku F1

# Create the Web App
az webapp create \
  --name {{appServices.api.name}} \
  --resource-group {{azure.resourceGroup}} \
  --plan {{appServices.api.name}}-plan \
  --runtime "DOTNETCORE:8.0"
```

Record the URL → `appServices.api.url`

#### 2. Enable Managed Identity

```bash
az webapp identity assign \
  --name {{appServices.api.name}} \
  --resource-group {{azure.resourceGroup}}
```

Note the `principalId` — you'll need it for Key Vault and Storage RBAC (Runsheets 3 & 4).

#### 3. Configure CORS

```bash
az webapp cors add \
  --resource-group {{azure.resourceGroup}} \
  --name {{appServices.api.name}} \
  --allowed-origins "https://{{appServices.portal.name}}.azurewebsites.net" "http://localhost:5173" "https://localhost:5173"
```

> **Why platform-level CORS?** Azure App Service runs IIS as a reverse proxy. IIS intercepts OPTIONS preflight requests before they reach ASP.NET Core. The `az webapp cors add` command tells IIS to add CORS headers. App-level CORS in `Program.cs` handles local dev (Kestrel, no IIS).

### Part B: Portal App Service

#### 1. Create the Web App

```bash
# Create App Service Plan (can share with API or create separate)
az appservice plan create \
  --name {{appServices.portal.name}}-plan \
  --resource-group {{azure.resourceGroup}} \
  --location {{azure.location}} \
  --sku F1

# Create the Web App (Node runtime for static file serving)
az webapp create \
  --name {{appServices.portal.name}} \
  --resource-group {{azure.resourceGroup}} \
  --plan {{appServices.portal.name}}-plan \
  --runtime "NODE:20-lts"
```

Record the URL → `appServices.portal.url`

### Part C: Grant RBAC Roles to API Managed Identity

After the API App Service is created, go back and complete:

1. **Key Vault** (Runsheet 4, Step 6): `Key Vault Secrets User` role
2. **Storage Account** (Runsheet 3, Step 6): `Storage Table Data Contributor` + `Storage Blob Data Contributor` roles (each storage service requires its own role)

### Part D: Update App Registration Redirect URIs

1. Go to Entra External ID tenant → **App registrations** → `{{projectName}}Portal`
2. **Authentication** → Under **Single-page application**, add:
   - `https://{{appServices.portal.name}}.azurewebsites.net`

## Deployment

### Manual API Deployment

```bash
# Build
dotnet publish {{projectName}}Api -c Release -o ./publish

# Zip (from the project subfolder)
cd publish && powershell -Command "Compress-Archive -Path * -DestinationPath ../deploy.zip -Force" && cd ..

# Deploy
az webapp deploy --resource-group {{azure.resourceGroup}} --name {{appServices.api.name}} --src-path deploy.zip --type zip

# Configure CORS (always re-run after deployment)
az webapp cors add --resource-group {{azure.resourceGroup}} --name {{appServices.api.name}} --allowed-origins "https://{{appServices.portal.name}}.azurewebsites.net" "http://localhost:5173"
```

### Manual Portal Deployment

```bash
cd {{projectName}}-portal

# Build
npm run build

# Copy web.config for IIS SPA routing
cp web.config dist/

# Zip
cd dist && powershell -Command "Compress-Archive -Path * -DestinationPath ../portal-deploy.zip -Force" && cd ..

# Deploy
az webapp deploy --resource-group {{azure.resourceGroup}} --name {{appServices.portal.name}} --src-path portal-deploy.zip --type zip
```

## Verification

- [ ] API App Service created and accessible (Swagger UI at `/swagger`)
- [ ] Portal App Service created and accessible
- [ ] API managed identity enabled
- [ ] CORS configured at platform level (API)
- [ ] Portal redirect URI added to Entra app registration
- [ ] RBAC roles assigned (Key Vault + Storage Table + Storage Blob)

## Common Issues

- **CORS errors in browser**: Always a symptom, not the cause. Check if the API is actually running (`curl` the endpoint directly). See lesson learned #4.
- **IIS 404 "resource removed" page**: Files deployed in wrong directory structure. Check Kudu VFS: `az rest --method get --url "https://<app>.scm.<region>.azurewebsites.net/api/vfs/site/wwwroot/" --resource "https://management.azure.com/"`
- **Portal shows blank page**: Missing `web.config` in deployed files. IIS needs it for SPA routing.

## Debugging Commands

```bash
# Check deployed files
az rest --method get --url "https://{{appServices.api.name}}.scm.<region>.azurewebsites.net/api/vfs/site/wwwroot/" --resource "https://management.azure.com/"

# Enable logging
az webapp log config --resource-group {{azure.resourceGroup}} --name {{appServices.api.name}} --application-logging filesystem --level error --detailed-error-messages true

# Download logs
az webapp log download --resource-group {{azure.resourceGroup}} --name {{appServices.api.name}} --log-file logs.zip

# Check CORS config
az webapp cors show --resource-group {{azure.resourceGroup}} --name {{appServices.api.name}}
```

## Output → `project-config.json`

```json
{
  "appServices": {
    "api": {
      "name": "{{appServices.api.name}}",
      "url": "https://{{appServices.api.name}}.azurewebsites.net"
    },
    "portal": {
      "name": "{{appServices.portal.name}}",
      "url": "https://{{appServices.portal.name}}.azurewebsites.net"
    }
  }
}
```
